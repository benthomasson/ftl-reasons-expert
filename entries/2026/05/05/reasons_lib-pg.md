# File: reasons_lib/pg.py

**Date:** 2026-05-05
**Time:** 15:16

# `reasons_lib/pg.py` — PostgreSQL Storage Backend

## 1. Purpose

This file implements a PostgreSQL-native storage backend for the TMS (Truth Maintenance System) dependency network. It is the multi-tenant, concurrent-writer alternative to the SQLite-based `storage.py`. Where the SQLite backend loads the full network into memory, `PgApi` executes each operation as an isolated SQL transaction against PostgreSQL — no in-memory `Network` object is ever constructed.

It owns: schema definition, CRUD for nodes/justifications/nogoods, truth value propagation, what-if simulation, dialectical operations (challenge/defend), search (with Postgres full-text search), explain/trace, compaction summaries, and access-tag-based visibility filtering.

## 2. Key Components

### Schema (module-level constants)

**`SCHEMA`** defines five tables, all partitioned by `project_id` (UUID) for multi-tenancy:

| Table | Role |
|---|---|
| `rms_nodes` | Belief nodes with truth value (IN/OUT), metadata (JSONB), source provenance |
| `rms_justifications` | SL/CP justifications linking a node to antecedents and outlist nodes |
| `rms_nogoods` | Recorded contradictions between node sets |
| `rms_propagation_log` | Append-only audit trail of all mutations |
| `rms_network_meta` | Key-value store (currently just `next_nogood_id`) |

**`INDEXES`** adds GIN indexes on `antecedents` and `outlist` JSONB columns (enabling `@>` containment queries) and a GIN full-text search index on `rms_nodes.text`.

### `PgApi` class

The single public class. Constructed with a connection string (or existing `psycopg` connection) and a `project_id`. Tracks whether it owns the connection (`_owns_conn`) to manage lifecycle correctly.

**Core mutations:**

- **`add_node`** — Inserts a node, parses SL/CP justification strings, validates all referenced nodes exist, inherits access tags from antecedents, computes initial truth value, and logs the action. Returns a dict with node type (premise vs derived) and premise count.
- **`add_justification`** — Adds a justification to an existing node, recomputes truth, and propagates changes.
- **`retract_node`** — Sets a node to OUT, marks `_retracted` in metadata, and propagates. If already OUT, only updates metadata (idempotent for the cascade).
- **`assert_node`** — Restores a retracted node to IN, clears the `_retracted` flag, and propagates.

**What-if simulation:**

- **`what_if_retract`** / **`what_if_assert`** — Execute the mutation inside a transaction, collect the cascade effects, then **rollback** unconditionally (via `try/finally: self.conn.rollback()`). Returns the hypothetical retracted/restored nodes with cascade depth and dependent counts. This is a read-only simulation that leaves the database unchanged.

**Dialectical operations:**

- **`challenge`** — Creates a new premise node and adds it to the target's outlist. If the target was IN, it may flip to OUT (the challenge defeats it). Wraps `_challenge_internal`.
- **`defend`** — Challenges a challenge node (meta-level): creates a defense node that, when IN, suppresses the challenge, potentially restoring the original target.

**Read operations:**

- **`get_status`** — Returns all nodes with justification counts.
- **`show_node`** — Full detail for one node including justifications, dependents, and metadata.
- **`search`** — Postgres `to_tsvector`/`plainto_tsquery` full-text search plus ILIKE fallback on node IDs. Includes 1-hop neighbor expansion.
- **`list_nodes`** — Filtered listing (by status, premises-only, has-dependents, namespace prefix).
- **`list_gated`** — Finds OUT nodes that *would* flip IN if their outlist blockers were retracted (GATE pattern).
- **`compact`** — Generates a token-budgeted markdown summary of the entire belief state, suitable for LLM context windows.
- **`explain_node`** — Recursive trace of why a node is IN or OUT, following justification chains.
- **`trace_assumptions`** — Finds all premise ancestors of a derived node.

**Nogoods:**

- **`add_nogood`** — Records a contradiction. If all involved nodes are IN, triggers dependency-directed backtracking: finds the least-entrenched culprit premise and retracts it.
- **`find_culprits`** — Identifies which premises, if retracted, would resolve a nogood.

## 3. Patterns

**Transaction-per-operation**: Every public method opens a cursor, does work, then calls `self.conn.commit()`. The connection is created with `autocommit=False`, so each method is an implicit transaction. On exception in `__exit__`, it calls `rollback()`.

**Rollback-based simulation**: `what_if_retract`/`what_if_assert` mutate real rows inside a transaction, collect results, then rollback. This reuses the production propagation code without duplicating logic.

**BFS propagation**: `_propagate` uses a `deque`-based BFS. It batch-fetches dependents, batch-fetches their justifications and all referenced truth values, then evaluates each dependent. Changed nodes are enqueued for the next wave. This is the same algorithm as the in-memory TMS but executed against SQL.

**JSONB for variable-length references**: Antecedents and outlist are stored as JSONB arrays rather than junction tables. This simplifies the schema at the cost of needing GIN indexes and `@>` containment queries for reverse lookups.

**Multi-tenancy via composite keys**: Every table uses `(id, project_id)` or includes `project_id` in its primary key. All queries filter by `project_id`. No cross-tenant data leakage is possible at the query level.

**Access-tag-based visibility**: Nodes carry `access_tags` in metadata. The `_is_visible` method enforces that the caller's `visible_to` set must contain all of a node's tags. Tags are inherited from antecedents when a derived node is created.

**Entrenchment scoring**: `_entrenchment` computes a numeric score for backtracking priority: premises score higher than derived nodes, sourced nodes higher than unsourced, and nodes with more dependents are more entrenched. The least-entrenched culprit gets retracted first.

## 4. Dependencies

**Imports:**
- `json` — serializing/deserializing JSONB columns
- `collections.deque` — BFS queue in `_propagate` and `_cascade_depth_pg`
- `datetime` — timestamps for log entries and node creation
- `psycopg` (optional, v3) — the PostgreSQL driver. Gracefully degrades to `None` with a clear error via `_require_psycopg()`

**Depended on by:**
- `tests/test_pg.py` — test suite for this module
- `tests/conftest.py` — likely provides fixtures that instantiate `PgApi`

The `.venv` entries in "Imported By" are false positives from the dependency scanner — `psycopg` internals don't import this module.

## 5. Flow

A typical mutation flow (e.g., `retract_node`):

1. Open cursor, fetch current truth value and metadata
2. If already OUT, update metadata only, commit, return early
3. Set truth to OUT, stamp `_retracted` in metadata
4. Log the retraction
5. Call `_propagate(cur, node_id)`:
   - BFS finds all nodes whose justifications reference the changed node (via `_find_dependents`)
   - For each dependent, re-evaluate all its justifications against current truth values
   - If truth changes, update the row, log it, enqueue for next BFS wave
   - Return `(went_out, went_in)` lists
6. Commit the transaction
7. Return summary dict

For `add_node` with justifications:
1. Check node doesn't already exist (raises `ValueError` if duplicate)
2. Insert node row
3. Parse SL/CP strings into justification dicts
4. Validate all referenced antecedent/outlist nodes exist (raises `KeyError` if not)
5. Insert justification rows
6. Inherit access tags from antecedents
7. Compute initial truth via `_compute_truth` (check if any justification is satisfied)
8. Update truth value, log, commit

## 6. Invariants

- **Node uniqueness**: `add_node` explicitly checks for duplicates before insert (belt-and-suspenders with the PK constraint).
- **Referential integrity**: `_validate_refs` ensures all antecedent/outlist node IDs exist before inserting justifications. The FK constraint on `rms_justifications` also enforces this at the DB level.
- **Justification types**: CHECK constraint limits `type` to `'SL'` or `'CP'`.
- **Truth values**: CHECK constraint limits `truth_value` to `'IN'` or `'OUT'`.
- **Premises are always IN unless explicitly retracted**: Nodes without justifications default to `'IN'`. `_compute_truth` returns `'IN'` for nodes with no justifications. `_propagate` skips nodes marked `_retracted` or with no justifications.
- **What-if never commits**: The `finally: self.conn.rollback()` guarantees simulation is side-effect-free.
- **Propagation terminates**: BFS tracks `visited` to prevent cycles. Each node is evaluated at most once per propagation wave.
- **Access tags are monotonically inherited**: A derived node's tags are the union of its own tags and all antecedent tags. Tags are never removed by inheritance.

## 7. Error Handling

- **Missing psycopg**: `_require_psycopg()` raises `ImportError` with install instructions. Called in `__init__`.
- **Duplicate nodes**: `add_node` raises `ValueError`.
- **Missing nodes**: `show_node`, `retract_node`, `assert_node`, `add_justification` raise `KeyError`.
- **Dangling references**: `_validate_refs` raises `KeyError` listing all missing node IDs.
- **No justification provided**: `add_justification` raises `ValueError` if both `sl` and `cp` are empty.
- **Access denied**: `show_node` raises `PermissionError` if visibility check fails.
- **Transaction safety**: `__exit__` calls `rollback()` on any exception. Normal paths call `commit()` explicitly. There is no retry logic — callers are expected to handle transient DB errors.
- **Metadata parsing**: Every metadata read includes `if isinstance(meta, str): meta = json.loads(meta)` as a defensive guard against psycopg returning raw strings vs parsed JSONB.

---

## Topics to Explore

- [file] `reasons_lib/storage.py` — The SQLite counterpart; comparing the two reveals which operations benefit from Postgres-native features (GIN indexes, JSONB, `plainto_tsquery`) vs the in-memory approach
- [function] `reasons_lib/pg.py:_propagate` — The BFS truth-propagation engine is the core algorithm; understanding its batch-fetch-then-evaluate pattern is essential for performance tuning
- [file] `tests/test_pg.py` — Shows how PgApi is tested (likely with a real Postgres instance or mock), and which edge cases (cycles, cascades, nogoods) are covered
- [function] `reasons_lib/pg.py:compact` — The token-budgeted summarizer is unusually complex for a storage layer; understanding its priority ordering (nogoods > OUT > IN by dependents) matters for LLM integration
- [general] `outlist-propagation-gap` — The CLAUDE.md documents a known issue where outlist nodes aren't tracked in the dependents index; `_find_dependents` does search both antecedents and outlist via `@>`, so this PG backend may not have the same bug as the in-memory version

## Beliefs

- `pg-what-if-always-rolls-back` — `what_if_retract` and `what_if_assert` unconditionally rollback via `try/finally`, guaranteeing no persistent side effects from simulation
- `pg-propagate-skips-retracted-nodes` — `_propagate` checks `meta.get("_retracted")` and skips nodes with that flag, preventing retracted premises from being re-activated by cascade
- `pg-find-dependents-searches-both-lists` — `_find_dependents` queries both `antecedents @> needle` and `outlist @> needle`, meaning outlist-dependent nodes ARE discovered during propagation (unlike the known issue documented for the in-memory backend)
- `pg-access-tags-inherited-as-union` — When a derived node is created, `_inherit_access_tags` merges access tags from all antecedents into the new node's tags using set union, and tags are never removed
- `pg-backtracking-prefers-least-entrenched` — `add_nogood` sorts culprit premises by entrenchment score (ascending) and retracts the first one, preferring to sacrifice premises with fewer dependents, no source provenance, and lower belief-type rank

