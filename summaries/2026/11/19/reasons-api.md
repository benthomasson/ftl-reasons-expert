---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/api.py

# `reasons/api.py` — Functional Python API for the RMS

## Purpose

This module is the **foundational Python surface** of the Reason Maintenance System (RMS). It provides stateless, functional entry points to every top-level operation the project supports — node CRUD, TMS propagation, search, export/import, staleness checking, Hive proposals, review/repair pipelines, deduplication, and more — without exposing callers to `Storage` lifecycle, argparse, or the underlying `Network` object graph.

Each public function:

1. Accepts a required set of arguments plus `db_path: str = DEFAULT_DB` and an optional `pg_conninfo, domain_id` pair for PostgreSQL delegation.
2. Opens the database, performs one logical operation, saves if write, closes.
3. Returns a **JSON-serializable dict** (or `str` for a handful of formatting functions).

It is effectively the "service layer" between the CLI/tests/MCP server and the lower-level `Storage`, `Network`, `pubsub`, `metadata`, and specialised modules (`derive`, `contradictions`, `cluster`, `check_stale`, `verify`, etc.).

---

## Key Components

### Constants / module state

- **`DEFAULT_DB = "reasons.db"`** — the default sqlite path shared across signatures.
- **`SOURCE_TYPES = {"code", "document", "self-description", "derived"}`** — enum enforced by `add_node`.
- **`_network_cache: dict[str, tuple[float, Network]]`** — process-level cache keyed by `db_path`. Value is `(mtime_at_load, Network)`.
- **`NEGATIVE_TERMS`, `NEGATIVE_BATCH_SIZE=50`, `NEGATIVE_CLASSIFY_PROMPT`** — parameters for `list_negative`.
- **`_STOP_WORDS`, `_TOPIC_STOP_WORDS`** — frozen sets used by `lookup` and `topics`.
- **`NEGATIVE_*` and other regexes** — parsing helpers.

### Core internal helpers

| Helper | Contract |
|---|---|
| `_is_visible(node, visible_to)` | Node visible iff `set(node.metadata.get("access_tags", [])) ⊆ visible_to`. No tags ⇒ always visible. |
| `_get_node_tags(db_path, node_ids)` | Reads `node_tags` table via raw sqlite; `node_ids=None` pulls all; `[]` returns `{}`. |
| `_filter_by_tags(ids, tag_filters, db_path)` | AND across filter keys, OR within each key. `"tag"` key = exact match; other keys auto-prefix `f"{key}:{value}"`. |
| `_resolve_namespace(id, ns)` | Prefixes only when `ns` truthy and `":"` not already in `id`. Assumes pre-namespaced ids come from another namespace (cross-namespace reference). |
| `_with_network(db_path, write=False)` | The primary load/operate/save context manager. See **Flow** below. |
| `clear_network_cache(db_path=None)` | Public invalidation used by tests and after external DB rewrites. |
| `_pg_dispatch(conninfo, domain_id, method, **kw)` | Lazily imports `PgApi`, calls `getattr(pg, method)(**kw)`. Central to the PG code path. |
| `_cascade_depth(net, target, retracted_id)` | BFS from `retracted_id` through `dependents` to `target`; returns shortest hop count (0 if unreachable). |

### Operation families (grouped by concern)

**Namespace / agent premises**
- `ensure_namespace(namespace, db_path)` — idempotently creates `{ns}:active` premise (metadata `role="agent_premise"`), which is the cascade gate for all beliefs under that namespace.
- `list_namespaces(...)` — scans for `*:active` nodes with `role="agent_premise"` and counts beliefs.

**Node CRUD & TMS**
- `init_db`, `add_node`, `add_justification`, `retract_node`, `assert_node`, `propagate`, `convert_to_premise`, `remove_justification`, `summarize`, `supersede`, `supersede_with_text`, `set_metadata`, `mark_duplicate`, `mark_superseded`.
- `what_if_retract`, `what_if_assert`, `what_if_supersede` — **non-mutating simulations** that `copy.deepcopy(net)` so the cached network is not corrupted.

**Query / explanation**
- `get_status`, `show_node`, `explain_node`, `trace_assumptions`, `trace_access_tags`, `find_culprits`, `get_belief_set`, `get_log`, `list_nodes`, `topics`, `list_gated`, `report_gated`, `report_belief`.

**Tags & sources** (SQLite only)
- `add_tags`, `remove_tags`, `get_tags`, `add_source` operate directly on the DB (not via `Network`), each raising `KeyError` if the node doesn't exist.

**Defeaters**
- `defeat_justification` — creates a premise defeater, appends it to a target justification's `outlist`, manually maintains `defeater.dependents.add(node_id)`, then `recompute_all()`. Returns only nodes whose truth value actually changed.
- `defeat_with_scope` — richer variant: creates scope premise nodes and a derived defeater citing them, so retracting any scope belief restores the original.
- `migrate_retract_to_defeaters`, `classify_defeat_reason_types`.

**Dialectical**
- `challenge`, `defend`, `add_nogood` — delegate to `Network.challenge/defend/add_nogood`.

**Search**
- `lookup(query, ...)` — simple all-terms search; progressive relaxation over word combinations when no results (`_MAX_RELAXATION=50`).
- `search(query, ...)` — FTS5-backed (porter stemming), with pipes for OR, `regex=True` for regex, and a `_substring_search` fallback when FTS returns nothing or is unavailable.
- Format helpers: `_format_markdown`, `_format_json`, `_format_minimal`, `_format_compact`, `_format_names_only`.

**Export / Import / HF / Mind Service**
- `export_network`, `export_markdown`, `export_card`.
- `import_beliefs`, `import_agent`, `sync_agent`, `import_json` (topological import with timestamp/source/tag restore), `import_hf`, `publish_hf`, `import_api`, `export_api`.

**Staleness / Integrity**
- `check_stale`, `check_integrity`, `backfill_hashes`, `hash_sources`, `pin_sources`, `pin_update`, `pin_lines`.

**Compact / Wiki**
- `compact(budget=500, truncate=True, ...)` — token-budgeted summary; `format="names-only"` bypasses the module and returns ID list.
- `build_wiki(...)` — groups via topics or semantic clusters.

**Review / Repair / Contradictions**
- `review_beliefs`, `review_justifications`, `review_premises`, `repair_premises`, `propose_update`, `store_llm_proposals`, `repair_smuggled`, `repair` (alias `research` downstream), `detect_contradictions`, `write_contradiction_plan`, `parse_contradiction_plan`, `apply_contradiction_plan`.

**Dedup / clustering**
- `list_clusters`, `deduplicate` (union-find over ID-token Jaccard or embedding cosine), `write_dedup_plan`, `parse_dedup_plan`, `apply_dedup_plan`, `_rewrite_dependents`.

**Hive proposals** — see **Patterns** below.
- `_proposals_db`, `_next_proposal_id`, `_stale_older_pending`, `propose_retraction`, `propose_supersession`, `propose_addition`, `propose_nogood`, `list_proposals`, `show_proposal`, `accept_proposal`, `reject_proposal`, `withdraw_proposal`.

**Derive & repos**
- `derive_prompt`, `derive_apply`, `add_repo`, `list_repos`.

---

## Patterns

### 1. Load / operate / save context manager (`_with_network`)
Every DB-touching function wraps logic in:

```python
with _with_network(db_path, write=True) as net:
    ...mutate net...
```

`_Ctx.__init__` decides whether to reuse the cache:
- Read the file's `mtime`.
- If cached `(mtime, network)` matches and `write=False` → reuse (no `Storage` opened).
- Otherwise → `Storage(db_path).load()`; `self.store` is non-None so `__exit__` will close it.

`__enter__` snapshots `{node_id: truth_value}` **only when writing and there are pubsub subscribers**.

`__exit__`:
- If `write` and no exception → `store.save(net)`, compute pubsub events from `_before`/`after`, `pubsub.publish`.
- `store.close()` if opened.
- Refresh cache **unless** `(write and exception)`, in which case the stale cache is preserved. This is the invariant that keeps the cache consistent with on-disk state.

### 2. Cache keyed by (db_path, mtime)
`_network_cache[db_path] = (mtime, net)`. Because `mtime` is taken after load, external mutations invalidate the cache naturally; `clear_network_cache()` is the escape hatch for tests and clock-granularity edge cases.

### 3. Simulation via `copy.deepcopy`
`what_if_retract`, `what_if_assert`, and `what_if_supersede` intentionally deep-copy the (possibly cached) `Network` before mutating, so the object stored in `_network_cache` remains semantically untouched. They also use `with _with_network(db_path, write=False)`, so even a "leaked" mutation cannot be persisted.

### 4. Dual-backend delegation
Almost every public function starts with:

```python
if pg_conninfo:
    return _pg_dispatch(pg_conninfo, domain_id, "<method>", ...)
```

Features not supported by the PG backend raise `NotImplementedError` **before** dispatching (e.g., `any_mode` in `add_node`, `tag` filtering in `search`, `min_depth` in `list_nodes`).

### 5. Modern Python signatures + JSON-safe returns
- Typed kwargs with defaults; `str | None` unions.
- Result dicts are small and stable: e.g. `retract_node` → `{"changed", "went_out", "went_in", "restoration_hints"}`.
- `_pg_dispatch` is called with the same **kwargs names** the sqlite implementations use, keeping signatures in lockstep with `PgApi`.

### 6. Namespace / agent premise pattern
`{ns}:active` is a normal premise node; it participates in every justification via `antecedents.insert(0, active_id)` in `add_node`. Retracting `{ns}:active` cascades all downstream beliefs OUT — the layer's structural way of implementing an agent kill switch, without special-casing the propagation engine.

### 7. Proposal staging (Hive pattern)
Proposals live in a separate `proposals` table (schema = `PROPOSALS_SCHEMA`), never mutating truth values at propose time. Lifecycle:

```
propose_*  →  pending ──accept_proposal──► accepted
              │       ──reject_proposal──► rejected
              │       ──withdraw_proposal► withdrawn
              │       ──_stale_older_pending / drift──► stale
```

Each `propose_*` calls `_stale_older_pending(conn, target_id, action, ...)` so **only the newest pending proposal for a `(target, action)` pair survives**. `accept_proposal` re-validates `truth_value`, `text_hash`, and `new_id` collisions against the live network before applying, and uses a `WHERE status = 'pending'` guard on the final UPDATE to survive concurrent acceptance races (returning `stale` on `rowcount==0`).

### 8. Search relaxation
Both `lookup` and `_fts_search` perform progressive term-dropping: if the full term set misses, iterate over subsets descending in size (bounded by `_MAX_RELAXATION` / `_MAX_RELAXATION_QUERIES = 50`) until a hit is found or the budget is exhausted.

### 9. Lazy imports for cyclic / optional deps
`from .pg import PgApi`, `from . import ask`, `from .cluster import …`, `import numpy as np`, and nearly all subpipeline imports are inside function bodies. This keeps `api.py` importable even without optional heavy deps installed (e.g., embeddings), and avoids cycles since `pg.py`, `derive.py`, `contradictions.py`, `repair.py`, `cli.py`, `ask.py`, `mcp_server.py` all import from `api.py`.

### 10. Union-find deduplication
`deduplicate` builds a `parent` map over all IN nodes and unions pairs above similarity threshold, then extracts clusters of size ≥ 2. `_rewrite_dependents` rewrites antecedent/outlist references to the survivor before retracting duplicates, keeping derived structures intact.

---

## Dependencies

### This module imports
- **In-tree**: `Storage`, `Network`, `Justification`, `Nogood`, `PROPOSALS_SCHEMA`, `pubsub`, `metadata.build_meta`, `PgApi` (lazy), plus pipeline-level helpers:
  - `derive` (`build_prompt`, `apply_proposals`, `_get_depth`, `_tokenize_id`, `_jaccard`), `check_stale` (`check_stale`, `hash_sources`, `pin_sources`, `pin_update`, `pin_lines`, `resolve_source_path`), `verify`, `report_belief`, `report_gated`, `import_beliefs`, `import_agent`, `hf`, `mind_service`, `compact`, `build_wiki`, `ask`, `llm`, `review`, `review_justifications`, `review_premises`, `repair`, `repair_premises`, `propose_update`, `contradictions`, `cluster`, `merkle`, `derive`.
- **Stdlib**: `copy, json, logging, os, re, sqlite3, sys, tempfile`, `collections.abc.Callable`, `collections.deque`, `collections.defaultdict`, `datetime`, `itertools.combinations`, `pathlib.Path`.
- **Third-party (optional)**: `numpy` (only in `deduplicate(semantic=True)`), `pyarrow`/embeddings inside `cluster`.

### Who imports this module
- **Every consumer**: `reasons/cli.py`, `reasons/mcp_server.py`, `reasons/ask.py`, `reasons/derive.py`, `reasons/contradictions.py`, `reasons/repair.py`, `reasons/pg.py`, and the mirrored `reasons_lib/api.py`.
- **The test suite**: 40+ test modules (`test_api.py`, `test_network_cache.py`, `test_pg_dispatch.py`, `test_access_tags.py`, …, `test_verify.py`).

So changes to any public function's key names, return keys, or exceptions ripple across both the CLI and MCP interface — treat the result dicts as a de facto public API.

---

## Flow

### Typical write path
```
caller (CLI/test)
   │
   ▼
public fn (e.g. add_node(..., db_path=..., pg_conninfo=None))
   │  if pg_conninfo → _pg_dispatch → PgApi.method()
   │
   ▼
build Justification(s) from comma-separated strings
   │
   ▼
with _with_network(db_path, write=True) as net:
   │  __init__:  if cache miss/stale → Storage.load()
   │             __enter__: if subscribers for db_path → snapshot _before
   │  body:      mutate net (add_node, retract, recompute_all, …)
   │  __exit__:  if no exception → store.save(net); compute_changes+publish
   │             store.close(); refresh _network_cache
   │
   ▼
return dict(result keys)
```

### Read path (fast case)
If `write=False` and `(os.path.getmtime(db_path), net)` matches the cache, no `Storage` instance is created at all — the operation runs purely against the cached `Network` object, which is safe as long as callers don't mutate it (why the `what_if_*` functions deep-copy first).

### PG path
`if pg_conninfo: return _pg_dispatch(pg_conninfo, domain_id, "method", **kwargs)`. Errors such as `NotImplementedError("any_mode is not supported with PostgreSQL")` are raised **before** dispatch, so they're deterministic across backends. Some exposed sub-behaviours (e.g., `depth > 1` in `search`, `tag` filtering in `list_nodes`) also refuse with `NotImplementedError`.

### Search path
```
search(query, regex?, "|"?, …)
  │
  ├─ regex=True       → _regex_search (Python re)
  ├─ "|" in query     → _or_search (union of _fts_search per term, then substring fallback per term)
  └─ default          → _fts_search → (fallback) _substring_search
  │
  ▼ filter (include_out, namespace, visible_to, tag)
  ▼ sort (newest|oldest|relevance)
  ▼ paginate (offset/limit)
  ▼ if format=="names-only": return
  ▼ BFS depth hops for neighbors along justifications + dependents
  ▼ _format_markdown / _format_json / _format_minimal / _format_compact
```

### Import JSON path (`import_json`)
Topologically sorts nodes so antecedents exist before dependents; adds each node, then explicitly restores `truth_value` via `retract`/`assert_node` and stamps timestamps **afterwards** so they aren't clobbered. After the `_with_network` block exits (saving the network), it reopens sqlite to restore `node_tags` and `node_sources` rows separately (those tables aren't part of the `Network` object).

---

## Invariants

1. **Namespace prefixing is idempotent.** `_resolve_namespace(id, ns)` only prepends `{ns}:` when `ns` is truthy *and* `id` has no `:`. Pre-namespaced ids are assumed to come from another namespace and left intact.
2. **`{ns}:active` is a premise.** Auto-created by `add_node(namespace=...)` and `ensure_namespace` with metadata `{"agent": ns, "role": "agent_premise"}`. Any justification for a namespaced node has this premise prepended.
3. **Access-tag visibility is subset-based, not intersection-based.** A node is visible iff `set(access_tags) ⊆ visible_to`. Nodes without tags are visible to everyone. Enforced by `_is_visible` and consistently applied to `matched_ids` *and* `neighbor_ids` in `search`.
4. **`SOURCE_TYPES` is closed.** Providing an out-of-set `source_type` to `add_node` raises `ValueError` before any DB write.
5. **`add_justification` requires one of `sl`, `cp`, `unless`.** Else `ValueError("Must provide --sl, --cp, or --unless")`. `outlist`-only justifies an `SL` with empty antecedents.
6. **`any_mode` splits one SL into N justifications, one antecedent each.** `premise_count` returned is `1` per justification (last-wins). PG does not support `any_mode`.
7. **`mark_duplicate` and `mark_superseded` refuse self-reference.** `source_id == canonical_id` (or `old_id == new_id`) raises `ValueError`.
8. **`propose_nogood` requires ≥ 2 distinct node ids** and all must currently exist. Node ids are sorted, joined by `-`, and used as `target_id`.
9. **Exactly one pending proposal per `(target_id, action)` pair.** `_stale_older_pending` stales the rest before inserting the new one. Proposal IDs are `prop-{target_id}-{action}-{count+1}`.
10. **`accept_proposal` re-validates before applying.** It checks node existence, current truth value equals `snapshot["truth_value"]`, and — if present — `node.text_hash == snapshot["text_hash"]`. Any drift → `_mark_stale(reason)` returning `{"applied": False, "status": "stale"}` without mutating the network. Final UPDATE uses `WHERE status = 'pending'` as an optimistic lock.
11. **`what_if_*` never mutate the DB.** Even a codec bug would be contained by `write=False` + `copy.deepcopy`.
12. **`init_db` will not clobber an existing DB without `force=True`.** `FileExistsError` otherwise; `force=True` deletes the file first.
13. **Cache mutation on write failure is suppressed** (`if not (write and exc_type is not None)` in `__exit__`), so failed writes leave the cache pointing at the pre-write state.

---

## Error Handling

- **`KeyError`** — missing node/record. Raised from `show_node`, `explain_node`, `trace_assumptions`, `trace_access_tags`, `set_metadata`, `mark_duplicate`, `mark_superseded`, `defeat_justification`, `propose_*`, `show_proposal`, `reject_proposal`, `withdraw_proposal`, tag/source helpers, and `what_if_*` on unknown ids.
- **`ValueError`** — semantically invalid input: bad `source_type`, out-of-range justification index, nonexistent `(target, action)` proposal state ("is 'accepted', not pending"), a proposal with no `node_ids`, self-referential `mark_duplicate`, `propose_nogood` with < 2 ids, `deduplicate` etc.
- **`PermissionError`** — access-tag failure in `show_node`, `explain_node`, `trace_assumptions`, `trace_access_tags` when `visible_to` is provided and the node's `access_tags` are not a subset.
- **`FileNotFoundError`** — `import_beliefs`, `import_agent`, `sync_agent`, `import_json`, `import_hf`.
- **`FileExistsError`** — `init_db` without `force`.
- **`NotImplementedError`** — PG-unsupported feature combos (e.g., `any_mode` with PG in `add_node`/`add_justification`, `depth > 1` in PG `search`, tag filtering in PG `search`, several filters in PG `list_nodes`, tag/source helpers when `pg_conninfo` is provided).
- **Swallowed / logged exceptions**:
  - `_with_network.__exit__` never suppresses exceptions (returns `False`), but if `exc_type is not None` it skips saving *and* skips cache update — so a caller-thrown exception leaves DB and cache consistent.
  - `verify_belief` catches failures from `retract_node` and timestamp stamping, populating optional keys `retract_failed` and `stamp_failed` rather than crashing.
  - `export_api` logs per-node push failures via `logger.warning` and increments an `errors` counter.
  - `report_belief` catches `KeyError`/`PermissionError` when fetching premise detail and substitutes `truth_value: "UNKNOWN"`.
  - `repair_premises`/`repair` log failures to `stderr` and rewrite the result's `action` to `"error"` rather than raising.
  - `apply_contradiction_plan` accumulates errors into `{"errors": [...]}` rather than aborting.
- **Progress feedback on long operations** (LLM-backed): `review_beliefs`, `review_premises`, `list_negative`, and `detect_contradictions` print to `stderr`; `on_batch` / `on_result` callbacks are passed through to sub-pipelines for streaming progress to the caller.
- **Results-over-exceptions pattern**: most write and review functions return a dict that may contain error-related keys (`"errors"`, `"failed"`, `"skipped"`, `"retract_failed"`, `"stamp_failed"`, `"applied": 0`) so callers can decide policy. Genuine programming/contract errors (bad ids, unsupported combos) still raise.

---

## Topics to Explore

- [file] `reasons/network.py` — The `Network` class that `_with_network` wraps; every mutation (`add_node`, `retract`, `assert_node`, `recompute_all`, `supersede`, `challenge`, `defend`, `add_nogood`, `explain`, `trace_assumptions`) ultimately lives here — understanding it is required to reason about "went_out vs went_in" and outlist semantics.
- [file] `reasons/storage.py` — Owns persistence and `PROPOSALS_SCHEMA`; the mtime-based cache in `_with_network` assumes `Storage.save` is atomic and that mtime is a reliable invalidation key. Also the source of `node_tags` / `node_sources` schemas read via raw sqlite.
- [file] `reasons/pubsub.py` — `_with_network` only snapshots `_before` when `pubsub.has_subscribers(db_path)`, and calls `pubsub.compute_changes` + `pubsub.publish` on every successful write. Read this to understand event ordering guarantees and exactly which subscribers trigger the extra diff work.
- [file] `reasons/pg.py` — The `PgApi` class is the mirror of this module for PostgreSQL; every `_pg_dispatch` call maps to a method there. Reading it will reveal which functions are truly feature-parity and which only have SQLite implementations.
- [function] `reasons/api.py:accept_proposal` — The most complex invariant-bearing function: re-validation against snapshots, stale-marking on drift, optimistic-lock UPDATE, and post-commit result reconciliation. Regression tests in `tests/test_proposals*` (via `store_llm_proposals`) are worth tracing.
- [general] `pubsub-change-computation` — How `compute_changes(before, after, db_path)` turns a full truth-value snapshot diff into events, and why the snapshot only happens when there are subscribers on the write path.

---

## Beliefs

- `network-cache-keyed-by-mtime` — `_network_cache` stores `(mtime, Network)` per `db_path`, and read-only calls (`write=False`) reuse the cached `Network` iff the current file mtime equals the cached one; otherwise a fresh `Storage(db_path).load()` is performed.
- `write-context-suppressed-on-exception` — `_with_network.__exit__` saves and publishes only when `write=True` and `exc_type is None`; on write exceptions it neither saves the network nor refreshes `_network_cache`, preserving on-disk/cache agreement.
- `namespaced-writes-prepend-active-premise` — `add_node(namespace=...)` prefixes the id via `_resolve_namespace` and inserts `{ns}:active` at position 0 of the first justification's `antecedents` (creating the premise node if missing), making retraction of `{ns}:active` a cascade kill switch.
- `accept-proposal-requires-snapshot-match` — `accept_proposal` marks a proposal `stale` and returns `{"applied": False}` whenever the live node's `truth_value` or `text_hash` no longer matches the proposal's `snapshot_json`.
- `search-falls-back-to-substring` — `search()` (default mode) first tries `_fts_search` against the `nodes_fts` FTS5 virtual table; if that returns no matches or raises `sqlite3.OperationalError`/`DatabaseError`, it falls back to `_substring_search` over node id and text.
- `what-if-simulations-never-persist` — `what_if_retract`, `what_if_assert`, and `what_if_supersede` open `_with_network(..., write=False)` and operate on a `copy.deepcopy` of the network, so the cached `Network` and the database are both left unmodified regardless of outcome.
