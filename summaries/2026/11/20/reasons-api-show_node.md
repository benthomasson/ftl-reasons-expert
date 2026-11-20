---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: show_node in reasons/api.py

# `show_node` — Detailed Explanation

## Purpose

`show_node` is the read-side "detail view" for a single belief in the Reason Maintenance System (RMS/TMS). It returns everything a UI, CLI, or LLM tool needs to understand one node: its text/truth state, the justifications that sustain it, the (reverse-indexed) nodes that depend on it, provenance metadata, and the auxiliary `node_tags` / `node_sources` rows that are not part of the serialized `Network`.

It exists as a standalone functional entrypoint so that callers (CLI, LangGraph tools, scripts) can load one node without managing a `Storage` object themselves. It sits next to `export_network` (whole-network dump) and `get_status`/`list_nodes` (list views), sharing the same `_with_network` load/cache machinery and the same `pg_conninfo` dispatcher.

## Contract

**Preconditions**
- A valid database exists at `db_path` (or the PG connection is live).
- `node_id` is looked up **exactly** — no namespace resolution, trimming, or alias handling happens here. If you have a namespace concept, the caller is responsible for passing the fully-qualified ID.
- `visible_to`, if provided, is the set of access tags the caller is authorized to see.

**Postconditions**
- On success, returns a JSON-serializable dict with **18 keys** (docstring only mentions 6). The keys are:
  `id`, `text`, `truth_value`, `supporting_justification`, `source`, `source_url`, `source_hash`, `text_hash`, `justifications`, `dependents`, `metadata`, `created_at`, `updated_at`, `reviewed_at`, `verified_at`, `retracted_at`, `tags`, `sources`.
- On failure, exactly one of `KeyError` (missing node) or `PermissionError` (access denied) is raised. `sqlite3` errors can propagate if the DB file is corrupted or locked.

**Invariants (assumed, not enforced by types)**
- The returned `metadata` dict and each `justifications[*].antecedents` / `.outlist` list are **direct references** into the cached `Network`. They are NOT deep-copied. Mutating them corrupts the in-process cache (see "Assumptions" below).
- The SQLite and PostgreSQL paths are expected to return a structurally identical dict (this is a caller-facing contract that the two backends must maintain).
- Node-level fields come from the cached `Network` while `tags` and `sources` come from the live SQLite file; these are *usually* consistent but the consistency is not guaranteed by the code.

## Parameters

| Param | Type | Notes |
|---|---|---|
| `node_id` | `str` | Exact match against `net.nodes`. Namespaces are *not* resolved here — caller must supply the final qualified ID. |
| `visible_to` | `list[str] \| None` | Access allow-list. `None` disables filtering entirely. `[]` matches only nodes whose `metadata.access_tags` is empty. A node is visible iff **all** its `access_tags` are in `visible_to` (see `_is_visible`). |
| `db_path` | `str` | Path to the SQLite file. Ignored when `pg_conninfo` is set. Serves both as the cache key in `_with_network` and as the target of a fresh `sqlite3.connect` for the tag/source queries. |
| `pg_conninfo` | truthy value (DSN dict/string) | If truthy, bypasses SQLite entirely and delegates to `PgApi.show_node`. |
| `domain_id` | any | Only consumed by the PostgreSQL path (tenant/namespace for the Pg schema). Ignored in the SQLite path. |

**Edge cases**
- `visible_to=[]` is a valid but restrictive setting — it hides every tagged node.
- A node with no `access_tags` is visible to everyone (including `visible_to=[]`).
- If the DB file on disk changes after `_with_network` loaded its cache (e.g., a concurrent write that doesn't bump mtime granularity), node fields could be stale relative to `tags`/`sources`.

## Return Value

A dict. Notable fields:

- **`supporting_justification`** — Bare pointer (typically an `int` index into `justifications`, or `None`). Not wrapped in a descriptive object here; consumers must interpret it themselves (see `_node_depth`, `export_network`).
- **`justifications`** — List of dicts, each `{type, antecedents, outlist, label, content_hash}`. `type` is `"SL"` (set-based/AND-OR in-list) or `"CP"` (contrapositive). `content_hash` is a Merkle hash for the justification (see `merkle` module).
- **`dependents`** — Sorted `list[str]` of reverse-edges (nodes whose justifications reference this one via `antecedents` or `outlist`). Sorted alphabetically, provided as a plain list.
- **`text_hash` / `source_hash`** — Merkle fields for integrity verification; may be empty strings if not backfilled.
- **`tags`** — Sorted list of tag strings from the `node_tags` table (ORDER BY tag).
- **`sources`** — List of dicts from `node_sources`, ordered by `rowid` (i.e., insertion order, not sorted), each with `{source_type, source_ref, source_url, source_hash, pinned_sha, pinned_lines, label, added_at}`.
- **Timestamps** — `created_at`, `updated_at`, `reviewed_at`, `verified_at`, `retracted_at` are returned raw (ISO-8601 strings or `""`).

Callers **must** handle `KeyError` and `PermissionError`. `supporting_justification` is untyped enough that callers should defensively bounds-check it before indexing into `justifications`.

## Algorithm

1. **Backend branch.** If `pg_conninfo` is truthy, immediately hand off to `_pg_dispatch(pg_conninfo, domain_id, "show_node", node_id=..., visible_to=...)`, which imports `reasons.pg.PgApi` and calls its `show_node` method inside a managed connection. The SQLite logic below is skipped entirely.
2. **Acquire the network.** `with _with_network(db_path) as net:` — this uses a module-level cache `_network_cache: dict[db_path, (mtime, Network)]`. On a hit (same `db_path`, unchanged mtime, read-only), the cached `Network` is reused. On a miss, `Storage(db_path).load()` is invoked. The context manager's `__exit__` **re-populates the cache even on read errors** (see Side Effects).
3. **Existence check.** `if node_id not in net.nodes: raise KeyError(f"Node '{node_id}' not found")`. This raises *before* any SQLite connection is opened.
4. **Visibility check.** If `visible_to is not None`, call `_is_visible(node, visible_to)`:
   - `tags = node.metadata.get("access_tags", [])` — if empty, return `True`.
   - Otherwise require every tag `t in tags` to be in `set(visible_to)`.
   If the check fails, raise `PermissionError(...)`.
5. **Build base result** from the `Node` object — id, text, truth value, source fields, hashes, justifications (list-comp mapping each `Justification` to a plain dict), dependents (sorted), metadata, and timestamps.
6. **Enrich from SQLite.** Open `sqlite3.connect(db_path)` — a *separate connection* from whatever `Storage` used. Execute two ordered SELECTs:
   - `SELECT tag FROM node_tags WHERE node_id = ? ORDER BY tag`
   - `SELECT source_type, ..., added_at FROM node_sources WHERE node_id = ? ORDER BY rowid`
   Both are assigned to `result["tags"]` / `result["sources"]`. `conn.close()` is in a `finally` block, so the connection is always released.
7. **Return** the assembled dict.

## Side Effects

- **Cache population.** Even on the failure paths (KeyError / PermissionError from within the `with` block), `_with_network.__exit__` runs and, because `write=False`, executes `_network_cache[db_path] = (mtime, self.network)`. Consequence: a failed read warms the cache for subsequent calls.
- **Fresh SQLite connection.** One `sqlite3.connect`, one open, two reads, close. This bypasses `Storage`/the cached `Network` — it reads the live DB file. This matters because tag/source rows are maintained outside the serialized `Network` by `add_tags` / `add_source` and friends.
- **No writes**, no save, no `pubsub.publish`.
- **Aliased references in the return value** (the biggest hidden side-effect hazard — see Assumptions).

## Error Handling

| Condition | Behavior |
|---|---|
| Node ID not in cached network | `KeyError("Node '...' not found")` |
| Access tags not subset of `visible_to` | `PermissionError("Node '...' requires access tags not in ...")` |
| DB file missing/corrupt | `sqlite3.OperationalError` / `sqlite3.DatabaseError` / `OSError` from `Storage(...).load()` or `os.path.getmtime` |
| `pg_conninfo` given, backend errors | Propagated from `PgApi` (contract is same exception types, but not guaranteed by this function) |
| Metadata `access_tags` is not a list | `_is_visible` iterates it; a non-iterable would raise `TypeError`. Not defended against. |

## Usage Patterns

Typical calls:

```python
detail = show_node("auth-uses-jwt")
detail = show_node("auth-uses-jwt", visible_to=["public", "internal"])
catch (KeyError, PermissionError) as e: ...
```

Common downstream callers in this file that rely on it: `build_wiki`, `report_gated`, `report_belief`, `verify_belief` — all of which wrap the call in `try/except (KeyError, PermissionError)` when iterating over a belt of node IDs that might have become unreadable.

**Caller obligations**
1. Catch `KeyError` and `PermissionError`.
2. **Do not mutate** the returned `metadata` dict or the `antecedents` / `outlist` lists inside `justifications`. These are live references into the cached `Network`; mutations would silently corrupt the cache until the next mtime-triggered reload.
3. Expect the same dict shape regardless of SQLite vs. PG backend.

## Dependencies

- `_with_network(db_path)` — cache + load/save context manager (`reasons/api.py`).
- `_is_visible(node, visible_to)` — access-tag predicate based on `node.metadata["access_tags"]`.
- `_pg_dispatch` + `reasons.pg.PgApi` — PostgreSQL backend.
- `Storage` / `Network` — from `reasons.storage` / `reasons.network`; provides the `Node` / `Justification` attributes read here (`text`, `truth_value`, `source`, `source_hash`, `text_hash`, `dependents`, `supporting_justification`, `content_hash`, timestamp fields).
- `sqlite3` (stdlib) — direct DB access for the `node_tags` / `node_sources` tables.
- Implicit: `DEFAULT_DB = "reasons.db"`.

## Assumptions not enforced by the type system

1. **Aliasing.** `result["metadata"] = node.metadata` and `"antecedents": j.antecedents` are not copies. Contrast with `export_network`, which snapshots `metadata` (`{k: v for k, v in n.metadata.items() if not k.startswith("_")}`) and re-lists antecedents via list comprehensions — those *do* copy. `show_node`'s resulting dict shares state with the cache.
2. **`supporting_justification` is a valid index.** Nothing verifies it is in `[0, len(justifications))`. Downstream code (`_node_depth`) explicitly bounds-checks, but `show_node` does not.
3. **Cache freshness.** `_with_network` invalidates only on file **mtime** change. Writes that land within the filesystem's mtime resolution window may not invalidate the cache; `show_node` would then return node fields from the pre-write snapshot while still reading fresh tags/sources from the live DB.
4. **`access_tags` shape.** `_is_visible` assumes `metadata["access_tags"]` is an iterable of strings; any other type raises at runtime, not at the type checker.
5. **Backend parity.** The SQLite return dict has 18 keys; the PG `PgApi.show_node` must match — this is an implicit cross-module contract not checked by tests on every field.
6. **`dependents` are direct-only.** The function returns immediate reverse edges only; transitive dependents must be computed elsewhere (`what_if_retract`, `_cascade_depth`).
7. **Tags are authoritative outside the network.** `node_tags` and `node_sources` are maintained by side-channel functions (`add_tags`, `add_source`, `import_json`, `import_hf`) — not by `Network.add_node`. So a node can have tags in SQLite that were never "seen" by the cached `Network`'s node object.

## Topics to Explore

- [function] `reasons/api.py:_with_network` — The caching + load/save context manager; explains why "read-only" `show_node` still populates state.
- [function] `reasons/api.py:_is_visible` — The access-tag subset rule that defines `PermissionError` semantics.
- [function] `reasons/api.py:export_network` — The whole-network analogue; useful for comparing which fields get copied vs. aliased.
- [function] `reasons/pg.py:PgApi.show_node` — The PostgreSQL counterpart that must return the same shape; check whether it honors `supporting_justification`, `content_hash`, etc.
- [general] `node_tags-and-node_sources-table-migration` — How tags/sources moved out of the serialized `Network` into side tables, and which write paths must stay in sync.

## Beliefs

- `show_node-returns-aliased-mutable-state` — The returned dict shares its `metadata` dict and each justification's `antecedents`/`outlist` list with the cached `Network`; mutating them corrupts the cache until the next mtime-invalidated reload.
- `visibility-skipped-when-no-access-tags` — `_is_visible` returns `True` for nodes without `access_tags` regardless of `visible_to`, so `visible_to=[]` blocks only tagged nodes.
- `show-node-dispatches-before-sqlite-touches-db` — When `pg_conninfo` is truthy, `show_node` returns from `_pg_dispatch` before opening the SQLite path at all; the SQLite-specific `sqlite3.connect(db_path)` and cache access are unreachable.
- `show-node-read-populates-cache-on-error` — The `_with_network` context manager re-stores `(mtime, network)` in `_network_cache` on `__exit__` even for read-only calls that raise `KeyError`/`PermissionError`, so a failed `show_node` can warm the cache for the next caller.
- `show-node-reads-tags-sources-outside-network-cache` — `tags` and `sources` are fetched via a fresh `sqlite3.connect(db_path)` rather than from the cached `Network`, so a concurrent commit to those tables is observed by `show_node` even if the `Network` snapshot is stale.
