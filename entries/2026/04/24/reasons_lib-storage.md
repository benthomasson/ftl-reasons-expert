# File: reasons_lib/storage.py

**Date:** 2026-04-24
**Time:** 16:56

## `reasons_lib/storage.py` — SQLite Persistence Layer

### 1. Purpose

This file owns the serialization boundary between the in-memory `Network` graph and a SQLite database on disk. It provides exactly two operations: **save** (full network → SQLite) and **load** (SQLite → full network). Every CLI command that mutates state goes through this layer via `api.py` to persist changes atomically.

### 2. Key Components

**`SCHEMA` (module constant)** — DDL for 6 tables plus one FTS virtual table:

| Table | Role |
|---|---|
| `nodes` | Belief nodes with truth value, provenance fields (`source`, `source_hash`, `date`), and a JSON metadata blob |
| `justifications` | SL justifications linking a node to its antecedents and outlist, ordered by `rowid` |
| `nogoods` | Recorded contradictions — sets of mutually incompatible nodes |
| `repos` | Named repository paths for source-hash staleness checking |
| `propagation_log` | Append-only audit trail of truth-value changes |
| `network_meta` | Key-value store for internal counters (currently just `next_nogood_id`) |
| `nodes_fts` | FTS5 full-text index over node IDs and text, powering `reasons search` |

**`Storage.__init__(self, db_path)`** — Opens the connection, enables WAL mode for concurrent readers, turns on foreign keys, and runs `_init_schema()` which is idempotent (`CREATE TABLE IF NOT EXISTS`).

**`Storage.save(self, network)`** — Delete-and-rewrite strategy inside a single transaction (`with self.conn:`). Clears all six tables, then reinserts every node, justification, nogood, repo, log entry, and the nogood counter. This is intentionally simple — it trades write performance for correctness on small-to-medium networks.

**`Storage.load(self) -> Network`** — Reconstructs a `Network` by:
1. Loading all node rows.
2. Loading all justification rows, grouping by `node_id`.
3. Building `Node` objects directly into `network.nodes` (bypassing `add_node` to avoid re-triggering propagation).
4. Calling `network._rebuild_dependents()` to reconstruct the reverse dependency index.
5. Loading nogoods, repos, metadata, and the propagation log.

**`Storage.close(self)`** — Closes the SQLite connection.

### 3. Patterns

- **Delete-and-rewrite**: Rather than diffing state, `save()` truncates and reinserts everything inside one transaction. This avoids complex upsert logic and guarantees the DB always matches the in-memory state exactly. The tradeoff is O(n) writes on every save — fine for belief networks that top out at hundreds or low thousands of nodes.

- **Bypass construction on load**: `load()` creates `Node` objects directly and assigns them to `network.nodes` rather than calling `network.add_node()`. This is critical — `add_node` would trigger propagation and dependent tracking during deserialization, corrupting the restored state. The dependent index is rebuilt separately via `_rebuild_dependents()`.

- **Defensive migration**: The `try/except` blocks around `network_meta` and `repos` reads let `load()` handle databases created before those tables existed. The fallback for `next_nogood_id` derives the counter from existing nogood IDs via regex, preventing ID collisions.

- **JSON-in-SQL**: Lists (antecedents, outlist, nogood node sets) and dicts (metadata) are stored as JSON text columns rather than normalized join tables. This simplifies the schema at the cost of queryability — but these fields are never queried individually.

### 4. Dependencies

**Imports:**
- `Node`, `Justification`, `Nogood` from `__init__.py` — the core dataclasses
- `Network` from `network.py` — the in-memory graph that gets serialized/deserialized
- `json`, `sqlite3`, `pathlib.Path` — stdlib
- `re` — imported lazily inside `load()` for nogood ID parsing

**Imported by:**
- `api.py` and `cli.py` — the two entry points that need persistence
- Multiple test files (`test_storage.py`, `test_dialectical.py`, `test_outlist.py`, etc.) — tests that exercise round-trip fidelity

### 5. Flow

**Save path**: `cli.py` → `api.py` (mutates a `Network` in memory) → `Storage.save(network)` → single SQLite transaction deletes all rows, reinserts everything → `conn.commit()` implicit in `with self.conn:`.

**Load path**: `cli.py` → `api.py` → `Storage.load()` → reads tables in dependency order (nodes first, then justifications, then nogoods/repos/log) → constructs `Network` with all state intact → returns `Network` for API to operate on.

The justification ordering matters: `ORDER BY rowid` in the justification query preserves insertion order, which determines which justification is considered "primary" when a node has multiple justifications.

### 6. Invariants

- **Atomicity**: `save()` runs inside `with self.conn:`, so a crash mid-save rolls back to the previous consistent state. This is the whole reason SQLite was chosen over markdown files.
- **FTS consistency**: The FTS index is always rebuilt from scratch during `save()` — it's a derived index, never the source of truth.
- **No partial loads**: `load()` either returns a complete `Network` or fails. There's no streaming or lazy-loading.
- **Justification ordering preserved**: `ORDER BY rowid` on load matches insertion order on save, maintaining the ordering semantics the network relies on.
- **Dependent index is derived**: `_rebuild_dependents()` reconstructs the reverse index from justifications on every load — the dependent index is never persisted.

### 7. Error Handling

Error handling is minimal and intentional:

- **Schema creation**: `IF NOT EXISTS` makes `_init_schema()` idempotent — safe to call on an already-initialized database.
- **Old database compatibility**: `try/except` around `network_meta` and `repos` reads silently handle missing tables from older schema versions. The `pass` on these exceptions is acceptable because the fallback behavior (derive counter from IDs, skip repos) is correct.
- **No validation on load**: `load()` trusts the database contents. If someone manually edits the SQLite file with invalid JSON in `antecedents_json`, it will raise `json.JSONDecodeError` without a friendly message. This is reasonable — the DB is an internal implementation detail, not a user-facing format.

## Topics to Explore

- [function] `reasons_lib/network.py:_rebuild_dependents` — The reverse-index reconstruction that `load()` depends on; understanding this clarifies why nodes are built directly rather than via `add_node`
- [file] `reasons_lib/api.py` — The functional API layer that wraps Storage.load/save around every mutation; shows the actual save/load lifecycle
- [file] `tests/test_storage.py` — Round-trip tests that define what "correct persistence" means for this system
- [function] `reasons_lib/network.py:add_node` — Compare with the bypass approach in `load()` to understand why propagation during deserialization would be wrong
- [general] `schema-migration-strategy` — The current `try/except` approach handles old-to-new, but there's no formal migration system; worth understanding the implications as the schema evolves

## Beliefs

- `storage-save-is-delete-rewrite` — `save()` deletes all rows and reinserts the entire network in a single transaction; it never diffs or upserts
- `storage-load-bypasses-add-node` — `load()` constructs Node objects directly into `network.nodes` without calling `add_node()`, avoiding propagation side effects during deserialization
- `storage-dependent-index-not-persisted` — The `dependents` reverse index is never stored in SQLite; it is rebuilt from justifications via `_rebuild_dependents()` on every load
- `storage-justification-order-preserved` — Justifications are loaded with `ORDER BY rowid`, preserving their original insertion order across save/load cycles
- `storage-old-schema-compat` — `load()` tolerates missing `network_meta` and `repos` tables via silent `try/except`, supporting databases created before those tables were added

