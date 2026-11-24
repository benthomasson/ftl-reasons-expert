---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: export_network in reasons/pg.py

## `PgApi.export_network(visible_to=None)`

### Purpose

Serializes the entire PostgreSQL-backed dependency network for a single `domain_id` into a plain Python dict that mirrors the JSON export schema (nodes, justifications embedded, nogoods, repos, meta). It is the read-half of the round-trip formed with `import_json`, and is the payload for `rms export`, backups, diffs, and cross-instance migration.

It exists because the Postgres backend deliberately avoids ever holding a full in-memory `Network` object; `export_network` is the one method that intentionally materializes the full state for a tenant, in a single read-only pass.

### Contract

- **Preconditions**: schema initialized (`rms_nodes`, `rms_justifications`, `rms_nogoods`, `rms_network_meta` exist — see `SCHEMA`); `self.conn` is a live `psycopg` connection; `self.domain_id` is a UUID string.
- **Postconditions**: returns a dict `{"meta": dict, "nodes": dict[str, dict], "nogoods": list[dict], "repos": dict[str, str]}`. Every visible node appears exactly once with an (empty or populated) `justifications` list. Metadata keys starting with `_` are stripped (private bookkeeping, e.g. `_retracted`).
- **Invariants**: no `commit()` or `rollback()` is issued → the connection's transaction state is unchanged. Because `__init__` passes `autocommit=False`, this method leaves the connection in the caller's current transaction (or opens an implicit one).
- **Do-not-leak invariant**: when `visible_to` is set, nogoods are *all-or-nothing* — a nogood is dropped if any of its referenced node IDs is missing from the visible set. Justification references are **not** re-filtered, but nodes are only fetched for nodes in the visible set (justifications are scoped to `ANY(visible node keys)`), so dangling-fk references cannot leak.

### Parameters

- `visible_to: Optional[Iterable[str]]`
  - If `None` (default): no visibility filtering — every node in the domain is exported.
  - If a collection (list/set of access-tag strings): delegated to `_is_visible(metadata, visible_to)`, which returns `True` when the node has **no** `access_tags`, or when **all** its tags are a subset of `visible_to` (union check via `all(t in visible_to ...)`).
  - **Edge cases**:
    - `visible_to=[]` → nodes with no tags are still visible (public), tagged nodes are excluded. This is not "deny all".
    - Passing a non-iterable will raise `TypeError` inside `_is_visible` — not validated.
    - A generator would be consumed once per node and then broken (silently disables visibility filtering after the first node). No coercion to `list` is done.

### Return Value

A dict with keys:

| Key | Type | Contents |
|---|---|---|
| `meta` | dict | Built by `reasons.metadata.build_meta(project_name, node_count, created_at)` from `rms_network_meta` (excluding `next_nogood_id`). |
| `nodes` | dict[str, dict] | Keyed by node ID. Each entry: `text`, `truth_value`, `justifications` (list of `{type, antecedents, outlist, label}`), `source`, `source_url`, `source_hash`, `date`, `metadata` (underscore-filtered), and ISO-8601 `created_at`/`updated_at`/`reviewed_at`/`verified_at`/`retracted_at` (`""` if null). |
| `nogoods` | list[dict] | Each: `{id, nodes, discovered, resolution}`. |
| `repos` | dict[str, str] | Repo name → path from `rms_network_meta` (keys `repo:<name>`). |

All lists are sorted by DB `id` (nodes: `ORDER BY id`; justifications: `ORDER BY id`; nogoods: `ORDER BY id`). Caller must not assume `nodes.values()` has insertion order matching anything else; keyed by ID.

### Algorithm

1. **Nodes query** — one `SELECT … FROM rms_nodes WHERE domain_id = %s ORDER BY id` fetching 13 columns. Iterate rows:
   - Defensive `json.loads(meta)` if `meta` came back as `str` (psycopg may return either `dict` or `str` depending on row factory/version — no `dict_row` is used here, so raw 13-tuples).
   - Skip node if `_is_visible` fails.
   - Store in `nodes[nid]` with `justifications=[]` and metadata filtered via `{k: v for k, v in meta.items() if not k.startswith("_")}`.
   - `_fmt_ts` converts timestamps (or falls back to `""` for `None`/`""`).

2. **Justifications (conditional)** — only if `nodes` is non-empty: `SELECT node_id, type, antecedents, outlist, label FROM rms_justifications WHERE domain_id = %s AND node_id = ANY(%s) ORDER BY id`, passing `list(nodes.keys())`. This is the *only* visibility-preserving step for justifications — a justification defaulting to an invisible ancestor would still refer to it in its `antecedents` array, but that ancestor is simply absent from the export; consumers must tolerate dangling references.
   - Defensive `if nid in nodes` — cannot trigger given the `ANY(keys)` filter, but cheap safety.

3. **Nogoods (unconditional)** — `SELECT id, nodes, discovered, resolution FROM rms_nogoods WHERE domain_id = %s ORDER BY id`. No `ANY()` filter (nogoods may reference arbitrary nodes). If `visible_to is not None`, keep only if `all(n in nodes for n in ng_nodes)` — i.e., a nogood is dropped as soon as **one** referenced node is hidden. If `visible_to is None`, nogoods referencing non-existent nodes are included verbatim (no FK protects `rms_nogoods.nodes`).

4. **Repos** — delegate to `self.list_repos()`, which opens its **own** cursor and queries `rms_network_meta` for `key LIKE 'repo:%'`. **Not** visibility filtered — repo paths are tenant-wide.

5. **Meta** — open a second cursor (`cur2`), `SELECT key, value FROM rms_network_meta WHERE domain_id = %s`, skip `next_nogood_id` (per-network counter, intentionally not exported), then call `build_meta(...)` with three clients. All other `rms_network_meta` rows (e.g. `schema_version`, `updated_at`) are silently discarded — the export's `meta` comes from `build_meta`, not from un-filtered stored metadata.

6. **Return** the composite dict. Cursors closed via context managers; connection state untouched.

### Side Effects

- **Reads** (4 SQL round-trips: nodes, justifications, nogoods, `rms_network_meta`; plus 1 inside `list_repos`).
- **No commits / rollbacks.** With `autocommit=False` this leaves a transaction open — a long-running caller who never commits will hold an idle-in-transaction session. This is returned to the caller's responsibility.
- **No writes, no I/O, no external calls.**
- **No caching** — always hits the DB.
- **No `commit()` even after read-only** — periodic repeated export calls accumulate implicit transactions; a caller-side `self.conn.commit()` or context-manager exit typically handles this.

### Error Handling

- **Not caught**: `psycopg` errors (connection loss, syntax error, permission) propagate to the caller.
- **Not caught**: `json.loads` if `metadata`/`antecedents`/`outlist`/`nogood.nodes` arrive as a `str` containing invalid JSON — impossible with JSONB columns but not guaranteed by types.
- **Not caught**: `TypeError` from `_is_visible` if `visible_to` is non-iterable.
- **Not caught**: `AttributeError` if `build_meta` is imported but signature changes.
- **Defensive**: `isinstance(meta, str)` guards JSONB round-trip quirks; `if nid in nodes` guards justification attachment.
- **`stored_meta.get(key, default)`** — always supplied with `""` defaults, so missing rows in `rms_network_meta` never raise.

### Usage Patterns

- **Read-only export**: `api.export_network(visible_to=["public", "team-a"])` → serialize to JSON, ship to a peer instance / write to a file.
- **Round-trip**: consumer is `import_json(data)` on another instance; message shape must match that function's expectations (`meta`, `nodes[*].justifications[*]`, `nogoods[*]`, plus timestamps used by `import_json`'s truth-value fixup pass).
- **Caller obligations**:
  1. Call `api.conn.commit()` (or use the context manager) after if you hold a long-lived connection.
  2. Treat `nodes`/`justifications`/`nogoods` as unordered-at-the-API-level (sorted by `id` incidentally).
  3. If `visible_to` is provided, do not assume the resulting export is internally reference-closed — justifications may reference node IDs that were filtered out. `import_json` tolerates this by topological add + defer.
  4. Do not rely on `meta` containing raw `rms_network_meta` rows — only `project_name`, `created_at`, `node_count`, and whatever `build_meta` synthesizes (`schema_version`, etc.).

### Dependencies

- **`psycopg` v3** — cursors and JSONB auto-deserialization.
- **`json`** — fallback parsing when JSONB arrives as string.
- **`reasons.metadata.build_meta`** — imported lazily inside the function to avoid a module-load cycle.
- **`self.list_repos()`** (`reasons.pg.PgApi.list_repos`) — reads `rms_network_meta` again.
- **`self._is_visible(metadata, visible_to)`** — access-tag semantics.
- **`self._fmt_ts`** — local nested function, NOT a method — duplicated inline in `show_node`.

### Assumptions not enforced by types

1. `visible_to` is re-iterable (a list/set), not a generator — a generator would silently break visibility after the first node.
2. `metadata` is always a JSON object (not a JSON array/string) — `meta.items()` in `nodes[nid] = {...}` would raise on a non-dict.
3. `self.domain_id` remains a valid UUID string throughout.
4. `rms_nogoods.nodes` references the same `domain_id` — no cross-domain check is performed.
5. `build_meta` accepts `project_name`, `node_count`, `created_at` (keyword-only, per the import).
6. Timestamp columns return `datetime | None | ""`; `_fmt_ts` handles `None`/`""`/`datetime`, but not e.g. `bytes`.
7. `list_repos` runs in the same transaction as the export — if a caller wraps this in an outer transaction and another writer interleaves, repos and nodes are *not* guaranteed consistent (but both are read-only so they reflect the same MVCC snapshot).

## Topics to Explore

- [function] `reasons/metadata.py:build_meta` — Understand the exact meta schema produced from the three stored fields; important for round-trip compatibility with `import_json`.
- [function] `reasons/pg.py:list_repos` — How repo name→path mappings are stored in `rms_network_meta` and whether they're domain-scoped or tenant-shared.
- [function] `reasons/pg.py:import_json` — The consuming half of the export contract; verify which fields it reads (e.g. `updated_at`, `reviewed_at`) to confirm nothing is silently dropped.
- [function] `reasons/pg.py:_is_visible` — The exact access-tag union semantics (public when no tags, subset match otherwise).
- [file] `tests/test_pg.py` — Existing coverage of `export_network`'s visibility filter and round-trip with `import_json`.

## Beliefs

- `export-network-strips-underscore-metadata` — Node `metadata` fields whose keys start with `_` (e.g. `_retracted`) are excluded from export output.
- `export-network-drops-nogood-if-any-node-invisible` — With `visible_to`, a nogood is dropped entirely if *any* of its referenced IDs is invisible — no partial nogoods leak.
- `export-network-does-not-commit` — The method issues no `commit()`/`rollback()`; callers must manage transaction lifecycle on the shared connection.
- `export-network-justifications-batched-by-node-id` — The justifications query is scoped by `node_id = ANY(%s)` over the visible node keys, avoiding a full-table scan of justifications.
- `export-network-repos-fetched-via-list-repos` — Repos are retrieved by calling `self.list_repos()`, so a second `rms_network_meta` scan happens outside the main cursor.
