---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Function: save in reasons/storage.py

# `Storage.save(network)`

## Purpose

`save` is the **write-side of the persistence layer**. It takes an in-memory `Network` object and rewrites the entire SQLite database so it fully matches that network. It is a *full-replace* strategy: “nuke and pave” every persisted table, then reinsert from the in-memory model. It exists so the on-disk DB always reflects a consistent snapshot of the network (nodes, justifications, nogoods, repos, propagation log, meta), including derived artifacts like the FTS5 search index.

## Contract

**Preconditions**
- `self.conn` is an open, schema-initialized connection (done in `__init__`/`_init_schema`).
- The DB has all the tables referenced (`nodes`, `justifications`, `nogoods`, `repos`, `propagation_log`, `network_meta`, `node_tags`, `node_sources`).
- `network` is a `Network` whose attributes (`nodes`, `nogoods`, `repos`, `log`, `meta`) are populated in-memory.

**Postconditions**
- Every table (except `proposals`, which is out of scope) matches the in-memory model.
- `nodes_fts` is a freshly rebuilt FTS5 index over `(id, text)`.
- External tag/source rows (written by `add_tags`/`add_source` directly to the DB) are preserved for any node that still exists in the network, and dropped for nodes removed from the network.
- `network_meta` contains at least `schema_version`, `project_name`, `created_at`, `updated_at`; `updated_at` is set to “now” (UTC, seconds).
- If any step raises, the surrounding `with self.conn:` context rolls the transaction back to its prior state.

**Invariants**
- Foreign keys are ON (`PRAGMA foreign_keys=ON` in `__init__`), so children (`justifications`, `node_tags`, `node_sources`) must be deleted before `nodes`.
- The FTS table is a virtual table (not FK-linked), so it is dropped/recreated rather than `DELETE`d.

## Parameters

- `network: Network` — the source of truth. The method reads:
  - `network.nodes.values()` → `Node.id`, `.text`, `.truth_value`, `.supporting_justification`, `.source`, `.source_url`, `.source_hash`, `.text_hash`, `.date`, `.metadata` (dict), `.created_at`, `.updated_at`, `.reviewed_at`, `.verified_at`, `.retracted_at`, `.justifications` (list of `Justification` with `.type`, `.antecedents`, `.outlist`, `.label`, `.content_hash`).
  - `network.nogoods` → items with `.id`, `.nodes`, `.discovered`, `.resolution`.
  - `network.meta` → dict (values are `str()`ed).
  - `network.repos` → dict of `name → path`.
  - `network.log` → list of dicts with `timestamp`, `action`, `target`, `value`.

**Edge cases**
- Empty network: everything is deleted, then only `network_meta` is repopulated.
- Node with `source == ''`/falsy: no `node_sources` row is created from the in-memory node (external rows may still restore it).
- `node.metadata` missing `access_tags`/`tags`/`source_type`/`pinned_sha`/`pinned_lines`: `.get(..., default)` supplies sensible defaults.
- Duplicate tags: `INSERT OR IGNORE` on `PRIMARY KEY (node_id, tag)` swallows them.

## Return Value

Returns `None`. There is no error/success return value. On failure the caller must catch the propagated exception (see below).

## Algorithm

1. **Open transaction** via `with self.conn:` — commit on success, rollback on exception.
2. **Snapshot external state**:
   - `ext_tags = SELECT node_id, tag FROM node_tags`
   - `ext_sources = SELECT node_id, source_type, source_ref, source_url, source_hash, pinned_sha, pinned_lines, label, added_at FROM node_sources`
   These capture rows written **outside** the in-memory model (e.g., `add_tags`/`add_source` per the comment) so they survive the wipe.
3. **Wipe everything**, children before parents (FK safety):
   - `DELETE node_tags`, `DELETE node_sources`, `DELETE justifications`
   - `DROP TABLE IF EXISTS nodes_fts`, then `CREATE VIRTUAL TABLE nodes_fts USING fts5(...)` (rebuild index; the drop is the “delete”)
   - `DELETE nodes`, `DELETE nogoods`, `DELETE repos`, `DELETE propagation_log`, `DELETE network_meta`
4. **Insert nodes + FTS + justifications**:
   - One row per node into `nodes` (all listed columns, `metadata` JSON-serialized).
   - One row into `nodes_fts (id, text)` per node.
   - N rows into `justifications` (antecedents/outlist JSON-serialized).
5. **Sync derived tag/source tables** from `node.metadata`:
   - `access_tags` → stored as `access:<tag>` in `node_tags`.
   - `tags` → stored verbatim in `node_tags`.
   - If `node.source` truthy → insert into `node_sources` (7 columns: no label/added_at) and record `(node.id, node.source)` in `written_sources`.
6. **Restore external rows** for surviving nodes:
   - `node_ids = set(network.nodes.keys())`
   - Each `ext_tags` row inserted with `INSERT OR IGNORE` if `nid in node_ids`.
   - Each `ext_sources` row inserted **only if** `nid in node_ids` **and** `(nid, source_ref) not in written_sources` — i.e., an in-memory source with the same `(node_id, source_ref)` takes precedence. This insert includes `label`/`added_at`.
7. **Insert nogoods** (one row each).
8. **Rebuild `network_meta`**:
   - `meta = dict(network.meta)` (copy)
   - `setdefault schema_version = SCHEMA_VERSION`
   - `setdefault project_name = self.db_path.stem` (note: differs from `__init__`, which prefers `self._project_name`; here the DB stem wins unless `network.meta` already carries `project_name`)
   - `setdefault created_at = now`
   - `meta["updated_at"] = now` (always overwritten)
   - `INSERT OR REPLACE` each key with `str(val)`.
9. **Insert repos** (name→path).
10. **Insert propagation_log** rows in list order.

## Side Effects

- **I/O**: Writes to the SQLite DB file (WAL mode is enabled on the connection).
- **State mutation**: The DB is fully replaced. Rows for nodes dropped from `network` are deleted from `node_tags` / `node_sources` / `justifications` / `nodes`.
- **Connection state**: The transaction is **committed** on success and **rolled back** on exception by the `with` block. Note the connection is *not* closed.
- **FTS rebuild**: The `nodes_fts` virtual table is dropped and recreated — this discards any FTS index state, which is intentional but expensive for large networks. Comment explicitly notes this is “simple strategy for small networks.”

## Error Handling

- No explicit `try/except` — **all exceptions propagate** to the caller.
- The `with self.conn:` context manager commits on clean exit and calls `conn.rollback()` on any exception, providing atomicity for the rewrite.
- Potential exception sources:
  - `sqlite3.IntegrityError` — e.g., FK violation from insertion ordering bugs, or `NOT NULL` violations.
  - `sqlite3.OperationalError` — schema mismatch (e.g., missing column in old DB not migrated), locked database.
  - `json.dumps` errors on un-serializable `metadata`/`antecedents`/`outlist`.
  - `AttributeError`/`KeyError` on malformed `network` or `network.log` entries (e.g., missing `timestamp` key).
- **Caveat**: DDL (`DROP`/`CREATE VIRTUAL TABLE`) participates in SQLite transactions from SQLite 3.6.8 onward, so rollback should restore the FTS table. This is an implicit assumption on the SQLite build.

## Usage Patterns

- Typically called by the CLI / API / agent layers after mutations: `storage.save(network)` — often right after `add_node`, propagation, or directive applications.
- Load/save is symmetric: `Storage.load()` reconstructs the network, callers mutate it in memory, then `save()` flushes. See `load()` for the read path it round-trips with.
- Callers must treat the whole call as **atomic but heavyweight** — for large networks this rewrites everything; batching mutations and saving once is the expected pattern.

## Dependencies

- **Internal**: `.Network` (type), `.Node`/`.Justification`/`.Nogood` (implicit via attribute access in `load`/`save`), `.metadata.SCHEMA_VERSION`.
- **Standard library**: `json`, `sqlite3` (via `self.conn`), `datetime`/`timezone` for `updated_at`.
- **Schema**: `SCHEMA` at module level (all tables), plus migration-added columns (`source_url`, `*_at`, `supporting_justification`, `text_hash`, `content_hash`) applied in `_init_schema`.

## Assumptions Not Enforced by the Type System

- `network.meta` values are `str()`-able; non-string values are silently coerced.
- `Node.metadata` is a JSON-serializable `dict`.
- Justification antecedents/outlist are JSON-serializable (e.g., lists of strings).
- `network.log` entries are dicts with exactly keys `timestamp`, `action`, `target`, `value` (a `KeyError` otherwise).
- `(node.id, node.source)` is the correct deduplication key for sources — a node with multiple sources in the same metadata won't be handled; only one `source` is captured per node.
- External tags/sources restoration assumes `node_ids` are the only authoritative set; rows for nodes not in `network` are intentionally dropped.
- `access_tags` in metadata are strings; they are prefixed with `"access:"` without validation.

## Topics to Explore

- [function] `reasons/storage.py:load` — The symmetric read path; verify it reconstructs exactly what `save` writes (round-trip invariant).
- [function] `reasons/storage.py:__init__` — Schema creation and migrations; explains why columns like `text_hash` exist even though not in `SCHEMA`.
- [file] `tests/test_storage.py` — Existing round-trip and edge-case coverage for save/load.
- [general] `external-tag-source-write-paths` — Wherever `add_tags`/`add_source` write directly to the DB, so you can reason about what `ext_tags`/`ext_sources` actually contain.
- [general] `fts5-tokenizer-config` — The `porter unicode61 tokenchars '-_'` tokenizer, and why FTS entries are rebuilt from scratch each save.

## Beliefs

- `save-is-transactional` — The entire rewrite runs inside `with self.conn:`; any exception triggers `rollback()`, leaving the previous DB state intact.
- `save-preserves-external-rows` — Tags/sources written directly to `node_tags`/`node_sources` are snapshotted before deletion and reinserted only for nodes still present in `network.nodes`.
- `save-rebuilds-fts-index` — `nodes_fts` is dropped and recreated on every save rather than incrementally updated.
- `save-is-full-replace` — The method deletes all rows in every targeted table before reinserting; it never does incremental writes.
- `save-dedupes-sources-on-node-and-source-ref` — An external `node_sources` row is skipped when `(node_id, source_ref)` matches an in-memory source written during the same save (`written_sources`).
- `save-requires-child-before-parent-deletion` — Because `PRAGMA foreign_keys=ON`, `justifications`/`node_tags`/`node_sources` are deleted before `nodes` to avoid FK violations.
