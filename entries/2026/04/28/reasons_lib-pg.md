# File: reasons_lib/pg.py

**Date:** 2026-04-28
**PR:** #50 (merged)
**Issue:** #46

## `reasons_lib/pg.py` — PostgreSQL-Native Storage Backend

### 1. Purpose

This file replaces the load-all/save-all pattern used by both `storage.py` (SQLite) and expert-service's `PgStorage` with row-level SQL transactions. Each API call is a single PostgreSQL transaction — no in-memory `Network` object is ever constructed. This enables concurrent writers and multi-tenant deployment.

The SQLite backend (`api.py` + `storage.py`) remains unchanged for CLI and single-user use. `pg.py` is a parallel API surface backed by PostgreSQL, intended for expert-service and multi-agent systems.

### 2. Architecture

`PgApi` is a class instantiated with `(conninfo, project_id)`. Every method executes SQL directly via psycopg v3. There is no intermediate `Network` object — truth values, justifications, and propagation all happen in SQL with Python-level BFS coordination.

```
PgApi(conninfo, project_id)
    │
    ├── add_node()      → INSERT + _compute_truth + commit
    ├── retract_node()   → UPDATE + _propagate (BFS) + commit
    ├── assert_node()    → UPDATE + _propagate (BFS) + commit
    ├── search()         → plainto_tsquery + GIN index + neighbor expansion
    └── add_nogood()     → INSERT + find_culprits + retract + _propagate + commit
```

### 3. Schema

Uses the same tables as expert-service (`rms_nodes`, `rms_justifications`, `rms_nogoods`, `rms_propagation_log`, `rms_network_meta`), all with composite primary keys `(id, project_id)` for multi-tenancy. Key difference from SQLite schema: JSONB columns instead of TEXT for antecedents, outlist, metadata, and nodes arrays.

Added GIN indexes on `rms_justifications.antecedents` and `rms_justifications.outlist` to accelerate the `@>` JSONB containment queries used by `_find_dependents`.

### 4. Propagation

The BFS propagation algorithm in `_propagate()` faithfully ports `network.py:_propagate()` (line 781) to SQL:

1. Drain the BFS queue into a batch
2. Find all dependents via JSONB containment: `antecedents @> '["node_id"]'::jsonb OR outlist @> '["node_id"]'::jsonb`
3. Batch-fetch all justifications and referenced node truth values for the dependents
4. Evaluate each dependent: ANY justification valid → IN, else OUT
5. UPDATE changed nodes, enqueue for further propagation
6. Skip nodes with `metadata->>'_retracted' = 'true'`

This batched approach minimizes SQL round-trips compared to per-node queries.

### 5. What Was Implemented (v0.20.0)

| Method | Category | Notes |
|--------|----------|-------|
| `init_db()` | Schema | CREATE TABLE IF NOT EXISTS + indexes |
| `add_node()` | Mutation | INSERT + justifications + access tag inheritance + truth computation |
| `add_justification()` | Mutation | Add justification to existing node + re-evaluate + propagate |
| `retract_node()` | Mutation | SET OUT + `_retracted` flag + cascade |
| `assert_node()` | Mutation | SET IN + clear `_retracted` + cascade |
| `get_status()` | Read | All nodes with truth values and justification counts |
| `show_node()` | Read | Full node details + justifications + dependents |
| `search()` | Read | `plainto_tsquery` + GIN + ILIKE fallback + neighbor expansion |
| `list_nodes()` | Read | Filtered by status, premises_only, has_dependents, namespace |
| `get_log()` | Read | Propagation audit trail |
| `add_nogood()` | Nogood | Record contradiction + dependency-directed backtracking |
| `find_culprits()` | Nogood | Trace to least-entrenched premises |
| `explain_node()` | Explain | Recursive justification chain trace |
| `trace_assumptions()` | Explain | Find all supporting premises |

57 tests in `tests/test_pg.py`, all marked `@pytest.mark.pg` and skipped without `DATABASE_URL`.

### 6. What Was NOT Implemented (Deferred)

These methods exist in `api.py` but are not yet ported to `PgApi`:

**Simulation:**
- `what_if_retract(node_id)` — read-only retraction simulation (no DB changes)
- `what_if_assert(node_id)` — read-only assertion simulation

**Dialectical operations:**
- `challenge(target_id, reason)` — create challenge premise, add to target's outlist
- `defend(target_id, challenge_id, reason)` — create defense that neutralizes challenge
- `supersede(old_id, new_id)` — add new_id to old_id's outlist (reversible replacement)
- `convert_to_premise(node_id)` — strip justifications, make node a premise

**Namespace support:**
- `ensure_namespace(namespace)` — create `{namespace}:active` premise
- `list_namespaces()` — list all namespace prefixes
- `add_node(..., namespace=...)` — auto-prefix node ID, depend on namespace premise
- `add_node(..., any_mode=True)` — expand SL into one justification per antecedent (OR semantics)

**Import/Export:**
- `export_network()` — full JSON export (nodes + nogoods)
- `export_markdown()` — beliefs.md format
- `export_compact()` — token-budgeted summary
- `import_json()` — lossless JSON import
- `import_beliefs()` — beliefs.md import
- `import_agent()` / `sync_agent()` — namespaced agent belief import/sync

**Maintenance:**
- `deduplicate()` — Jaccard similarity clustering of similar beliefs
- `check_stale()` / `hash_sources()` — source file staleness detection
- `derive_prompt()` / `derive_apply()` — LLM-assisted belief derivation

**Utility:**
- `get_belief_set()` — list all IN node IDs
- `summarize()` — create summary node over a group
- `trace_access_tags()` — union of tags in dependency subgraph

**Filter parameters not yet supported in `list_nodes()`:**
- `challenged` — filter by nodes with active challenges
- `min_depth` / `max_depth` — filter by derivation depth

**Infrastructure (not per-method):**
- CLI integration — no `--backend pg` flag or `DATABASE_URL` handling in CLI
- Async wrapper — `PgApiAsync` using `psycopg.AsyncConnection`
- expert-service migration — replace `PgStorage` load/save with `PgApi` incremental ops
- RBAC query filtering — `WHERE access_tags && ARRAY[:user_tags]` in SQL
- Semantic search — pgvector embeddings
- `_find_dependents` batching — current N+1 query pattern could be a single query

### 7. Design Decisions

**psycopg v3 (not psycopg2 or SQLAlchemy):** Modern, native JSONB support, sync+async from same codebase. Optional dependency — CLI users never need it.

**Application-level BFS (not stored procedures):** Matches reference implementation exactly, easier to verify correctness, no PL/pgSQL maintenance burden. Can optimize to recursive CTEs later if profiling shows need.

**No materialized dependents table:** Dependents computed on-the-fly via JSONB containment queries against GIN indexes. Avoids sync issues. Can add materialized table later as optimization.

**One transaction per API call:** Each public method commits at the end. `__exit__` rolls back on exception. Connection can be caller-provided for batch operations.

### 8. Code Review Notes

Two rounds of multi-model review (Claude + Gemini). Gemini flagged 4 "BROKEN" issues in both rounds — all were false positives matching the reference `network.py` implementation:

1. BFS `visited` set — prevents infinite loops, same as `network.py:785`
2. `trace_assumptions` through all justifications — intentional, same as `network.py:228`
3. No propagation on `add_node` — same as `network.py:109`
4. `assert_node` unconditionally sets IN — that's what assert means in a TMS, same as `network.py:150`

Two valid issues from Claude were fixed before merge:
- `to_tsquery` with manual quoting → `plainto_tsquery` (handles user input with single quotes)
- `__exit__` added rollback on exception for non-owned connections
