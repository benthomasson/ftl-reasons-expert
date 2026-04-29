# File: reasons_lib/storage.py

**Date:** 2026-04-29
**Time:** 16:56

## `reasons_lib/storage.py` — SQLite Persistence Layer

### 1. Purpose

This file owns **serialization and deserialization of the entire `Network` object to/from a SQLite database**. It's the persistence boundary — everything above it (CLI, API, derive, check_stale) works with in-memory `Network` objects, and this module handles the round-trip to disk. It's the only module that touches SQLite directly for the core belief store (the sibling `pg.py` handles PostgreSQL for federation).

### 2. Key Components

**`SCHEMA` (module-level constant)** — DDL for seven tables plus one FTS virtual table:

| Table | Role |
|---|---|
| `nodes` | Belief nodes with truth value, provenance (`source`, `source_hash`, `date`), and arbitrary metadata |
| `justifications` | SL/CP justification records linking nodes to their antecedents and outlists |
| `nogoods` | Recorded contradictions (sets of mutually inconsistent nodes) |
| `repos` | Named repository paths (for multi-repo code-expert use) |
| `propagation_log` | Audit trail of truth-value changes during propagation |
| `network_meta` | Key-value store for counters like `next_nogood_id` |
| `nodes_fts` | FTS5 full-text index over node IDs and text, powering `reasons search` |

**`Storage` class** — Three-method interface:

- **`__init__(db_path)`** — Opens (or creates) the SQLite file, enables WAL mode and foreign keys, runs `_init_schema()`. WAL gives concurrent readers during long writes.
- **`save(network)`** — Writes the entire `Network` to SQLite using a **delete-and-rewrite** strategy. Wrapped in a `with self.conn:` context manager, so the full wipe-and-insert is atomic.
- **`load()`** — Reads all tables back into a fresh `Network` instance. Carefully avoids `Network.add_node()` to preserve exact persisted state (truth values, justifications) without triggering propagation logic.
- **`close()`** — Closes the connection.

### 3. Patterns

**Snapshot persistence (not incremental).** `save()` deletes all rows then reinserts everything. This is simple and correct for the expected scale (hundreds to low thousands of beliefs), but would need rethinking for larger networks.

**Bypass domain logic on load.** `load()` builds `Node` objects directly and assigns them into `network.nodes` rather than calling `add_node()`. This is intentional — `add_node()` would trigger propagation and truth-value computation, corrupting the persisted state. The dependent index is rebuilt afterward via `network._rebuild_dependents()`.

**Defensive loading with `try/except` pass.** Tables like `repos` and `network_meta` may not exist in databases created by older schema versions. The code catches exceptions and falls back gracefully rather than requiring migrations.

**JSON-in-columns.** Lists of antecedent IDs, outlist IDs, nogood node sets, and node metadata are stored as JSON text columns. This avoids junction tables at the cost of not being queryable via SQL joins — acceptable because the code always loads the full network.

### 4. Dependencies

**Imports:**
- `Node`, `Justification`, `Nogood` from `reasons_lib/__init__.py` — the domain dataclasses
- `Network` from `reasons_lib/network.py` — the in-memory graph with propagation logic
- `json`, `sqlite3`, `pathlib.Path` — stdlib
- `re` — imported lazily inside `load()` only when deriving `next_nogood_id` from existing nogoods

**Imported by:**
- `reasons_lib/api.py` and `reasons_lib/cli.py` — the two main entry points that open/save the database
- Eight test files covering storage round-trips, CLI integration, dialectical reasoning, nogood IDs, etc.

### 5. Flow

**Save path:**
```
CLI/API creates Network → mutates it → calls Storage.save(network)
  → DELETE all rows (within transaction)
  → INSERT nodes + FTS entries
  → INSERT justifications per node
  → INSERT nogoods
  → INSERT network_meta (next_nogood_id)
  → INSERT repos
  → INSERT propagation_log
  → COMMIT (implicit via context manager)
```

**Load path:**
```
Storage.load()
  → SELECT all nodes → build Node objects (no justifications yet)
  → SELECT all justifications ORDER BY rowid → group by node_id
  → Attach justifications to Node objects
  → Assign nodes into Network.nodes dict
  → network._rebuild_dependents()  ← reconstructs the reverse index
  → SELECT nogoods → append to network.nogoods
  → SELECT network_meta → restore next_nogood_id counter
      (fallback: scan existing nogood IDs for max)
  → SELECT repos → populate network.repos
  → SELECT propagation_log ORDER BY rowid → append to network.log
  → return Network
```

The `ORDER BY rowid` on justifications and log entries preserves insertion order, which matters for justification priority (first valid justification wins in the TMS) and log replay.

### 6. Invariants

- **Atomicity**: `save()` runs inside a single transaction. A crash mid-save leaves the old data intact (WAL mode ensures this).
- **Round-trip fidelity**: `load()` must reconstruct the exact `Network` state that was passed to `save()`, including truth values, justification order, nogood IDs, and the nogood counter. No propagation runs during load.
- **FTS sync**: The `nodes_fts` table is always rebuilt from scratch during `save()`, so it's guaranteed consistent with `nodes`.
- **Justification ordering**: `ORDER BY rowid` in the justification query preserves the original insertion order, matching the order in `node.justifications`.
- **Nogood ID monotonicity**: `next_nogood_id` is persisted and restored to prevent ID collisions after restart. The fallback derivation (`max existing ID + 1`) handles databases that predate the `network_meta` table.

### 7. Error Handling

Minimal and intentional:

- **Schema compatibility**: `CREATE TABLE IF NOT EXISTS` and `CREATE VIRTUAL TABLE IF NOT EXISTS` make `_init_schema()` idempotent — safe to run against an existing database.
- **Missing tables**: `load()` wraps `repos` and `network_meta` reads in bare `try/except Exception: pass` to handle databases from before those tables were added. This is a migration substitute — it silently ignores missing tables rather than failing.
- **No validation on load**: The code trusts the database contents. Malformed JSON in a column would raise `json.JSONDecodeError` and propagate uncaught.
- **No connection pooling or retry**: Single connection opened in `__init__`, expected to be used by one thread in a CLI context.

---

## Topics to Explore

- [function] `reasons_lib/network.py:_rebuild_dependents` — The reverse-index reconstruction called after load; understanding it clarifies why `load()` bypasses `add_node()`
- [file] `reasons_lib/network.py` — The in-memory `Network` class with propagation, retraction, and nogood detection — the domain model this module serializes
- [file] `tests/test_storage.py` — Round-trip tests that define what save/load fidelity means in practice
- [file] `reasons_lib/pg.py` — The PostgreSQL counterpart for federated/multi-agent scenarios; compare the persistence strategies
- [general] `delete-and-rewrite-vs-upsert` — The current snapshot strategy works for small networks but would need incremental upserts or change tracking at scale

## Beliefs

- `storage-save-is-atomic-snapshot` — `Storage.save()` deletes all rows and reinserts the full network within a single SQLite transaction; partial writes cannot occur
- `storage-load-bypasses-propagation` — `Storage.load()` constructs nodes directly without calling `Network.add_node()`, preserving persisted truth values without triggering TMS propagation
- `justification-order-preserved-by-rowid` — Justification insertion order is preserved across save/load via `ORDER BY rowid`, which matters for TMS justification priority
- `storage-fts-rebuilt-on-every-save` — The `nodes_fts` full-text index is deleted and rebuilt from scratch on each `save()`, guaranteeing consistency with the `nodes` table
- `storage-handles-schema-evolution-via-try-except` — Missing tables from older database versions (`repos`, `network_meta`) are handled by swallowing exceptions during load, not by formal migrations

