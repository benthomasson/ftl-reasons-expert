# File: reasons_lib/storage.py

**Date:** 2026-04-23
**Time:** 16:38

# `reasons_lib/storage.py` — SQLite Persistence Layer

## Purpose

This file is the serialization boundary between the in-memory `Network` (a dependency-directed graph of beliefs, justifications, and contradictions) and a SQLite database on disk. It owns exactly one responsibility: **bidirectional mapping between `Network` objects and relational tables**, with ACID guarantees so that propagation cascades (where retracting one belief cascades to its dependents) are atomic.

It does not contain any business logic — no truth maintenance, no propagation, no staleness checks. It's a dumb persistence layer that faithfully snapshots and restores the network's exact state.

## Key Components

### `SCHEMA` (module-level constant)

The DDL for six tables:

| Table | Role |
|---|---|
| `nodes` | Beliefs — each with an id, text, truth value (`IN`/`OUT`), provenance fields (`source`, `source_hash`, `date`), and a JSON metadata bag |
| `justifications` | Why a node is believed — links a node to its supporting antecedents (inlist) and negated premises (outlist), with a type discriminator and optional label |
| `nogoods` | Recorded contradictions between sets of nodes |
| `repos` | Tracked repository paths (for code-expert use) |
| `propagation_log` | Audit trail of truth-value changes |
| `nodes_fts` | FTS5 virtual table for full-text search over node ids and text |

Notable: `justifications` uses `AUTOINCREMENT` on rowid and the load path orders by `rowid` — this preserves insertion order, which matters because justification priority determines which justification "wins" during truth maintenance.

### `Storage` class

**Constructor** (`__init__`): Opens a SQLite connection, enables WAL mode for concurrent reads, turns on foreign key enforcement, and runs `_init_schema` (which is idempotent via `CREATE ... IF NOT EXISTS`).

**`save(network)`**: Destructive full-replace strategy — deletes all rows from every table, then re-inserts the entire network state. This is intentionally simple; the docstring acknowledges it's designed for "small networks." The entire operation runs inside a `with self.conn:` context manager, making it a single transaction.

**`load()`**: Reconstructs a `Network` from SQLite in a careful order:
1. Load all node rows
2. Load all justifications, grouped by `node_id`
3. Build `Node` objects directly into `network.nodes` (bypassing `Network.add_node` to avoid triggering propagation)
4. Rebuild the `dependents` index by walking every justification's antecedents and outlist
5. Load nogoods, repos, and the propagation log

**`close()`**: Closes the SQLite connection.

## Patterns

**Snapshot-and-restore**: Rather than incremental updates (which would require change tracking), `save` does a full wipe-and-rewrite. This trades write performance for simplicity and correctness — there's no drift between in-memory and on-disk state.

**Bypass-on-load**: `load()` writes directly to `network.nodes[nid]` instead of calling `network.add_node()`. This is deliberate — `add_node` would trigger truth maintenance propagation, but we're restoring a previously-computed state, not deriving a new one.

**JSON-in-SQL**: List-valued fields (antecedents, outlist, nogood node sets) and the metadata dict are stored as JSON text columns. This avoids join tables at the cost of not being queryable via SQL — acceptable because all queries go through the Python `Network` object, not raw SQL.

**Context manager for transactions**: `with self.conn:` in `save()` ensures the entire write is atomic. If any insert fails, the whole save rolls back.

## Dependencies

**Imports:**
- `Network` from `.network` — the in-memory graph this class persists
- `Node`, `Justification`, `Nogood` from `.__init__` — the domain dataclasses

**Imported by:**
- `reasons_lib/api.py` and `reasons_lib/cli.py` — the two entry points that create `Storage` instances to load/save networks
- Five test files for various integration tests

`Storage` has no awareness of truth maintenance, staleness checking, or any other domain logic. It's a leaf dependency — it imports domain types but nothing imports it except entry points and tests.

## Flow

### Save path
```
save(network)
  → BEGIN TRANSACTION (implicit via `with self.conn:`)
  → DELETE all rows from all 6 tables
  → for each node: INSERT into nodes + nodes_fts
    → for each justification on that node: INSERT into justifications
  → for each nogood: INSERT into nogoods
  → for each repo: INSERT into repos
  → for each log entry: INSERT into propagation_log
  → COMMIT
```

### Load path
```
load()
  → SELECT all nodes
  → SELECT all justifications ORDER BY rowid
  → group justifications by node_id
  → construct Node objects with their justifications attached
  → insert directly into network.nodes dict
  → rebuild dependents index (walk antecedents + outlist)
  → SELECT nogoods, repos, log → append to network
  → return network
```

## Invariants

1. **Justification ordering is preserved**: Justifications are loaded `ORDER BY rowid`, and `rowid` uses `AUTOINCREMENT`. Insertion order from `save()` matches the order in `node.justifications`, so round-tripping preserves justification priority.

2. **Dependents index is derived, not stored**: The `dependents` set on each node is rebuilt from justifications during `load()`. It's never persisted. This means it's always consistent with the justification data.

3. **Truth values are stored, not recomputed**: `load()` trusts the stored `truth_value`. It does not re-run propagation. This means the database is the source of truth for node status — if you edit the database directly, the loaded network will reflect those edits without validation.

4. **Full-text index is always in sync**: `nodes_fts` is wiped and rebuilt alongside `nodes` in every `save()`, so they can't drift.

## Error Handling

Minimal and intentional:

- **Schema creation**: `IF NOT EXISTS` makes `_init_schema` idempotent — safe to call on an existing database.
- **Repos table**: `load()` wraps the repos query in a bare `try/except Exception: pass`. This is a forward-compatibility shim — older databases may lack the `repos` table. The comment says so explicitly.
- **No validation on load**: If the database contains invalid JSON in a `_json` column, `json.loads` will raise. There's no defensive handling — this is correct because `save()` is the only writer, and it always writes valid JSON.
- **No connection pooling or retry**: Single connection, no reconnect logic. Appropriate for a CLI tool where each invocation opens, uses, and closes one database.

## Topics to Explore

- [file] `reasons_lib/network.py` — The in-memory `Network` class that Storage serializes; understanding its `add_node` method clarifies why `load()` bypasses it
- [function] `reasons_lib/network.py:propagate` — The truth maintenance algorithm whose results Storage persists but never invokes
- [file] `reasons_lib/api.py` — The primary consumer of Storage; shows how load/save bracket network mutations
- [file] `tests/test_storage.py` — Round-trip tests that verify save/load preserves network state exactly
- [general] `outlist-semantics` — How the outlist (negated premises) in justifications interacts with the dependents index rebuilt during load

## Beliefs

- `storage-save-is-full-replace` — `Storage.save()` deletes all rows from every table before re-inserting; there is no incremental/differential update path
- `storage-load-bypasses-propagation` — `load()` constructs nodes directly into `network.nodes` rather than calling `add_node`, so truth maintenance does not fire during deserialization
- `justification-order-preserved-via-rowid` — Justification insertion order is preserved across save/load cycles by using `AUTOINCREMENT` rowid and `ORDER BY rowid` on read
- `dependents-index-is-derived-on-load` — The `node.dependents` set is never persisted; it's rebuilt by walking all justification antecedents and outlists during `load()`
- `repos-table-load-is-fault-tolerant` — Loading the repos table is wrapped in a bare except to handle databases created before the repos table was added

