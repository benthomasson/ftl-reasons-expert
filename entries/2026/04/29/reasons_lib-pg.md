# File: reasons_lib/pg.py

**Date:** 2026-04-29
**Time:** 17:14

# `reasons_lib/pg.py` — PostgreSQL Storage Backend for the Truth Maintenance System

## 1. Purpose

This file implements a **PostgreSQL-native storage backend** for the `ftl-reasons` dependency network — a Truth Maintenance System (TMS) based on Doyle's 1979 design. While the project also has a SQLite backend (`storage.py`) and an in-memory network (`network.py`), this module provides the same logical operations directly against PostgreSQL, enabling **concurrent writers and multi-tenant isolation** via a `project_id` partition key.

It owns the full lifecycle: schema creation, node/justification CRUD, truth-value propagation, nogood detection with dependency-directed backtracking, dialectical challenge/defend operations, what-if simulation, search, and a token-budgeted compact summary.

## 2. Key Components

### Schema (lines ~22–72)

Five tables, all partitioned by `project_id UUID`:

| Table | Role |
|---|---|
| `rms_nodes` | Belief nodes with `id`, `text`, `truth_value` (IN/OUT), JSONB `metadata` |
| `rms_justifications` | SL or CP justifications linking a node to its `antecedents` (inlist) and `outlist` |
| `rms_nogoods` | Recorded contradictions between sets of nodes |
| `rms_propagation_log` | Append-only audit trail of all mutations |
| `rms_network_meta` | Key-value store for network-level state (e.g., `next_nogood_id`) |

Indexes include GIN indexes on `antecedents` and `outlist` JSONB arrays (for `@>` containment queries) and a full-text search index on `rms_nodes.text`.

### `PgApi` class

The single public class. Constructor accepts either a connection string or an existing `psycopg` connection, plus a `project_id` that scopes all queries.

**Core mutations:**

- **`add_node`** — Inserts a node, optionally with SL/CP justifications. Computes initial truth value from justification validity. Inherits access tags from antecedents. Returns node type (premise vs. derived) and current premise count.
- **`add_justification`** — Adds a new justification to an existing node. Recomputes truth and propagates changes.
- **`retract_node`** — Sets a node to OUT, marks `_retracted` in metadata, and propagates. Idempotent if already OUT.
- **`assert_node`** — Forces a node to IN, clears `_retracted` flag, and propagates.

**What-if simulation:**

- **`what_if_retract` / `what_if_assert`** — Perform the mutation inside a transaction, collect cascade effects, then **rollback**. Returns depth-annotated lists of what would change without actually changing anything.

**Dialectical operations:**

- **`challenge`** — Creates a new premise node and adds it to the target's outlist, potentially flipping the target to OUT. Implements argumentation-style defeat.
- **`defend`** — Counter-challenge: creates a challenge against an existing challenge node, potentially restoring the original target.

**Read operations:**

- **`get_status`** — Lists all nodes with truth values and justification counts.
- **`show_node`** — Full detail for one node including justifications and dependents.
- **`search`** — PostgreSQL full-text search (`to_tsvector`/`plainto_tsquery`) with ILIKE fallback on node IDs, plus 1-hop neighbor expansion.
- **`list_nodes`** — Filtered listing (by status, premises-only, has-dependents, namespace).
- **`list_gated`** — Finds OUT nodes that would flip IN if their outlist blockers went OUT (GATE pattern).
- **`explain_node`** — Recursive trace showing why a node is IN or OUT.
- **`trace_assumptions`** — Walks justification chains to find all leaf premises.
- **`compact`** — Token-budgeted summary of the entire belief state, suitable for LLM context windows.

**Nogood operations:**

- **`add_nogood`** — Records a contradiction. If all nodes are IN, performs dependency-directed backtracking: finds the least-entrenched culprit premise and retracts it.
- **`find_culprits`** — Identifies which premises could be retracted to resolve a contradiction, ranked by entrenchment score.

## 3. Patterns

**Transaction-per-operation.** Each public method opens a cursor, does all work, and calls `self.conn.commit()`. The connection is opened with `autocommit=False`, so each method is an implicit transaction. The `__exit__` handler calls `rollback()` on exceptions.

**Rollback-based simulation.** `what_if_retract`/`what_if_assert` use `try/finally: self.conn.rollback()` to perform real mutations, snapshot the results, then undo everything. This reuses the propagation engine without duplicating logic.

**BFS propagation.** `_propagate` does breadth-first traversal of the dependency graph. It batch-fetches dependents per BFS level, bulk-loads their justifications and referenced truth values into a `truth_cache`, then evaluates each dependent. Changed nodes get queued for the next BFS round.

**JSONB containment for reverse lookups.** `_find_dependents` uses `antecedents @> '["node-id"]'::jsonb` to find which justifications reference a given node. This leverages the GIN indexes.

**Multi-tenant isolation.** Every query includes `project_id = %s`. There are no cross-project queries. The primary keys are composite `(id, project_id)`.

**Entrenchment-based backtracking.** When a nogood is active, `_find_culprits_internal` scores each premise by a weighted heuristic (premise status +100, has source +50, source hash +25, dependent count ×10, belief type bonus). The least-entrenched premise gets retracted.

**Access tag visibility.** Nodes can carry `access_tags` in metadata. Read operations accept `visible_to` (a set of tags) and filter out nodes whose tags aren't a subset. Tags are inherited from antecedents during `add_node`.

## 4. Dependencies

**Imports:**
- `json` — JSONB serialization/deserialization
- `collections.deque` — BFS queue for propagation and cascade depth
- `datetime` — Timestamps for nodes and log entries
- `psycopg` (v3) — PostgreSQL driver, imported with a try/except so the module loads even without it installed

**Depended on by:**
- `tests/test_pg.py` — Test suite for this module
- `tests/conftest.py` — Likely provides a `PgApi` fixture
- `build/lib/reasons_lib/pg.py` — Build artifact (copy)

The `.venv/psycopg` entries in the "Imported By" list are false positives — those are psycopg's internal modules, not actual importers of this file.

## 5. Flow

A typical write operation like `retract_node`:

1. Open cursor, fetch current truth value and metadata
2. If already OUT, just update metadata and return (idempotent path)
3. Set truth_value to OUT, stamp `_retracted` in metadata
4. Log the retraction
5. Call `_propagate(cur, node_id)`:
   - BFS from changed node
   - `_find_dependents` identifies nodes referencing the changed node in their justifications
   - For each dependent, re-evaluate all justifications against current truth values
   - If truth value changes, update the row, log it, and queue the dependent for the next BFS round
6. Commit the transaction
7. Return `{changed, went_out, went_in}`

The `compact` method builds a token-budgeted summary:
1. Fetches all nodes, nogoods, dependency counts, and first justifications
2. Sorts IN nodes by dependent count (most-depended-on first)
3. Builds sections (Nogoods → OUT → IN), stopping when estimated token count hits budget
4. Summary nodes can hide their covered nodes to save space

## 6. Invariants

- **Node IDs are unique per project.** `add_node` checks for existence and raises `ValueError` on duplicates.
- **Justifications must reference existing nodes** (enforced at the FK level for `node_id`, but antecedent/outlist references are stored as JSONB arrays without FK constraints — they can reference nonexistent nodes, which default to truth value OUT via `truth_cache.get(a, "OUT")`).
- **Premises have no justifications.** A node with zero rows in `rms_justifications` is a premise and defaults to IN.
- **Derived truth is disjunctive.** A node is IN if *any* justification is valid (all antecedents IN and all outlist nodes OUT). It's OUT only if *all* justifications are invalid.
- **Retracted nodes stay OUT regardless of justifications.** `_propagate` skips nodes with `meta["_retracted"] == True`.
- **What-if operations never commit.** They always rollback, even on success.
- **Propagation is monotone within a single BFS pass** — visited nodes aren't re-evaluated (tracked via `visited` set).

## 7. Error Handling

- **Missing psycopg:** `_require_psycopg()` raises `ImportError` with install instructions at construction time.
- **Node not found:** Raises `KeyError` with the node ID.
- **Duplicate node:** Raises `ValueError`.
- **Access denied:** `show_node` raises `PermissionError` when visibility check fails.
- **No justification given:** `add_justification` raises `ValueError` if both `sl` and `cp` are empty.
- **Transaction safety:** `__exit__` calls `rollback()` on any exception. Individual methods commit only on the success path.
- **JSONB parsing:** Every metadata/antecedents/outlist field is defensively checked with `isinstance(x, str)` before `json.loads()`, because psycopg may or may not auto-deserialize JSONB depending on configuration.

---

## Topics to Explore

- [file] `reasons_lib/storage.py` — The SQLite backend; compare its API surface and propagation logic to understand what `PgApi` mirrors and where they diverge
- [file] `reasons_lib/network.py` — The in-memory network implementation; `PgApi` reimplements its algorithms in SQL rather than delegating to it
- [file] `tests/test_pg.py` — Shows how the PostgreSQL backend is tested (likely with a real or containerized Postgres), and covers edge cases in propagation and what-if
- [function] `reasons_lib/pg.py:_propagate` — The core BFS truth propagation engine; understanding its batching and caching strategy is essential for performance tuning
- [general] `outlist-dependent-tracking` — The known issue where outlist nodes aren't tracked in the dependents index applies here too — `_find_dependents` does search both antecedents and outlist, but the GATE re-evaluation problem (documented in CLAUDE.md) may still manifest in edge cases

## Beliefs

- `pg-api-transaction-per-method` — Every public method on `PgApi` commits its own transaction; there is no way to compose multiple operations into a single transaction from outside the class
- `pg-what-if-uses-rollback` — `what_if_retract` and `what_if_assert` perform real mutations then rollback, rather than simulating on a copy; this means they temporarily modify database state and are not safe under concurrent readers without MVCC isolation
- `pg-missing-antecedents-default-out` — Antecedent and outlist references in justifications are not foreign-key constrained; missing nodes are treated as OUT via `truth_cache.get(id, "OUT")`, which silently degrades rather than erroring
- `pg-retracted-flag-blocks-propagation` — Nodes with `_retracted: true` in metadata are skipped during propagation, meaning manual retraction is sticky even if justifications would otherwise support IN
- `pg-find-dependents-searches-both-lists` — `_find_dependents` checks both `antecedents` and `outlist` JSONB arrays using GIN-indexed containment queries, which means outlist-based dependencies are tracked here (unlike the known issue documented for the in-memory network)

