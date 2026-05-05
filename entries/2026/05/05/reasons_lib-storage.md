# File: reasons_lib/storage.py

**Date:** 2026-05-05
**Time:** 15:19

## Purpose

`storage.py` is the persistence layer for the belief network. It owns the SQLite serialization contract: how an in-memory `Network` (nodes, justifications, nogoods, repos, propagation log) gets written to and read from a `.db` file. Every other module that needs durable state goes through `Storage` — it's the single gateway between the live object graph and disk.

The module is used directly by `reasons_lib/api.py` (which wraps it for the CLI and higher-level operations) and by a large number of tests.

## Key Components

### `SCHEMA` (module-level constant)

The DDL for six tables plus one FTS virtual table:

| Table | Role |
|---|---|
| `nodes` | Belief nodes — id, text, truth value, provenance fields |
| `justifications` | SL/CP justifications linking a node to its antecedents and outlist |
| `nogoods` | Recorded contradictions (sets of mutually inconsistent nodes) |
| `repos` | Tracked git repositories (name → path) |
| `propagation_log` | Audit trail of truth-value changes |
| `network_meta` | Key-value store for internal counters (currently just `next_nogood_id`) |
| `nodes_fts` | FTS5 full-text index on node id + text, using Porter stemming |

All tables use `CREATE TABLE IF NOT EXISTS`, making schema init idempotent.

### `Storage` class

**`__init__(self, db_path)`** — Opens (or creates) the SQLite database, enables WAL mode for concurrent reads, turns on foreign keys, and runs schema init.

**`_init_schema(self)`** — Runs the DDL, then checks whether the `source_url` column exists on `nodes`. If not, it `ALTER TABLE ADD COLUMN`s it. This is the only migration path in the file — it handles databases created before the `source_url` field was added.

**`save(self, network)`** — Full snapshot write. Deletes all rows from every table, then re-inserts the entire network state. The FTS table is dropped and recreated (you can't `DELETE FROM` an FTS5 table cleanly in all SQLite versions). Runs inside `with self.conn:` so the entire write is a single transaction — either everything commits or nothing does.

**`load(self)`** — Reconstructs a `Network` from SQLite. The load order matters:
1. Nodes (without justifications)
2. Justifications (grouped by `node_id`, preserving `rowid` insertion order)
3. Assemble nodes with their justifications, inserting directly into `network.nodes` (bypasses `add_node` to avoid re-triggering propagation)
4. Rebuild the dependents index via `network._rebuild_dependents()`
5. Nogoods
6. `next_nogood_id` counter (from `network_meta`, with fallback derivation from existing nogood IDs)
7. Repos
8. Propagation log

**`close(self)`** — Closes the SQLite connection.

## Patterns

**Snapshot persistence (not incremental)** — `save()` wipes and rewrites everything. The docstring calls this "simple strategy for small networks." This avoids diff-tracking complexity but means save cost scales linearly with network size.

**Bypass-on-load** — `load()` writes directly to `network.nodes[nid]` instead of calling `network.add_node()`. This is intentional: `add_node` would trigger truth maintenance propagation, which would corrupt the stored state. The network is being *restored*, not *built*.

**Defensive backward compatibility** — Both `_init_schema` and `load` handle databases that predate the `source_url` column. `load()` also wraps `repos` and `network_meta` reads in `try/except` blocks for old databases missing those tables.

**WAL mode** — Enables write-ahead logging for better concurrent read performance (important since the CLI and tests may read while another process writes).

## Dependencies

**Imports:**
- `Node`, `Justification`, `Nogood` from `reasons_lib.__init__` — the domain model dataclasses
- `Network` from `reasons_lib.network` — the in-memory belief graph
- `json` — for serializing lists/dicts into JSON text columns
- `sqlite3`, `Path` — standard library

**Depended on by:**
- `reasons_lib/api.py` — the primary consumer; wraps `Storage` for all CLI operations
- ~10 test files that directly instantiate `Storage` for integration tests

## Flow

### Save path
```
caller → save(network)
  → BEGIN TRANSACTION (implicit via `with self.conn`)
  → DELETE all rows from all tables
  → DROP + recreate FTS table
  → INSERT each node + its justifications + FTS entry
  → INSERT each nogood
  → INSERT next_nogood_id into network_meta
  → INSERT repos
  → INSERT propagation log entries
  → COMMIT
```

### Load path
```
caller → load()
  → SELECT all nodes
  → SELECT all justifications (ordered by rowid)
  → group justifications by node_id
  → construct Node objects with justifications attached
  → assign directly to network.nodes dict
  → _rebuild_dependents()
  → SELECT nogoods → append to network.nogoods
  → SELECT next_nogood_id from network_meta (fallback: derive from max existing ID)
  → SELECT repos
  → SELECT propagation_log (ordered by rowid)
  → return fully hydrated Network
```

## Invariants

1. **Atomic saves** — The entire `save()` runs in one transaction. A crash mid-save won't leave a half-written database.
2. **Justification ordering is preserved** — Justifications are inserted in list order and loaded via `ORDER BY rowid`, so the order survives a round-trip. This matters because the first valid justification determines a node's truth value.
3. **Load bypasses propagation** — Nodes are placed directly into `network.nodes`, not via `add_node`. The dependents index is rebuilt from justifications after all nodes are loaded, not incrementally.
4. **FTS consistency** — The FTS table is dropped and recreated on every save, keeping it in sync with `nodes` by construction rather than by incremental updates.
5. **Nogood ID counter continuity** — `next_nogood_id` is persisted in `network_meta` and restored on load, with a fallback that scans existing nogood IDs to avoid collisions.

## Error Handling

Error handling is minimal and deliberate:

- **Schema migration** (`_init_schema`): The `ALTER TABLE` for `source_url` is protected by a column-existence check, not a try/except. If the ALTER fails for another reason, it raises.
- **`load()` backward compat**: `repos` and `network_meta` reads are wrapped in bare `except Exception: pass` blocks. This silently swallows errors if those tables don't exist (old databases). The `source_url` column check is done via `PRAGMA table_info` rather than try/except.
- **No validation on load**: `load()` trusts the database contents. It doesn't validate that antecedent node IDs actually exist, that truth values are valid, or that justification types are recognized. The assumption is that `save()` wrote valid data.
- **No connection pooling or retry**: One connection per `Storage` instance, no retry on `SQLITE_BUSY`.

## Topics to Explore

- [file] `reasons_lib/network.py` — The in-memory `Network` class whose state `Storage` serializes; understanding `_rebuild_dependents()` and `add_node()` explains why load bypasses propagation
- [function] `reasons_lib/api.py:ReasonsAPI` — The primary consumer of `Storage`; shows how save/load integrate into the CLI workflow
- [file] `tests/test_storage.py` — Round-trip tests that verify save/load fidelity, edge cases with empty networks, and backward compatibility
- [function] `reasons_lib/network.py:_rebuild_dependents` — The method called after load to reconstruct the dependents index from justification antecedents and outlists
- [general] `snapshot-vs-incremental-persistence` — The current wipe-and-rewrite strategy works for small networks but would need rethinking for large-scale use; worth understanding the tradeoff

## Beliefs

- `storage-save-is-atomic-snapshot` — `save()` deletes all rows and re-inserts the full network in a single SQLite transaction; there is no incremental/diff-based persistence
- `storage-load-bypasses-add-node` — `load()` assigns nodes directly to `network.nodes` and calls `_rebuild_dependents()` afterward, deliberately skipping `add_node` to avoid triggering propagation on restore
- `storage-justification-order-preserved` — Justifications are inserted in list order and loaded via `ORDER BY rowid`, preserving the ordering that determines which justification is evaluated first
- `storage-fts-rebuilt-on-every-save` — The `nodes_fts` virtual table is dropped and recreated on each `save()` rather than incrementally updated, ensuring it stays in sync with node data by construction
- `storage-nogood-counter-persisted` — `next_nogood_id` is stored in `network_meta` and restored on load, with a regex-based fallback that derives the counter from existing nogood IDs to prevent collisions

