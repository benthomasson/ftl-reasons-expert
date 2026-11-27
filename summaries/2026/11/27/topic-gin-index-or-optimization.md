---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Whether the `antecedents @> ... OR outlist @> ...` query actually uses both GIN indexes is a performance-relevant question worth benchmarking.

## Should you benchmark whether `antecedents @> ... OR outlist @> ...` uses both GIN indexes?

**Short answer:** Yes, this is worth benchmarking — but it's a question that *cannot* be answered from the static code alone. The observations confirm the indexes and the query exist, but they don't tell us what the PostgreSQL planner actually does at runtime.

### What the observations tell us

1. **Both GIN indexes exist.** In `reasons/pg.py` `INDEXES` (lines 85–86):
   - `idx_rms_justifications_antecedents ON rms_justifications USING gin(antecedents)`
   - `idx_rms_justifications_outlist ON rms_justifications USING gin(outlist)`
   These are created in `PgApi.init_db()` via `cur.execute(INDEXES)` (around `reasons/pg.py:84-86` in the schema block shown in `read_pg_storage_module`).

2. **The query in question** is at `reasons/pg.py:3270`:
   ```sql
   AND (antecedents @> %s::jsonb OR outlist @> %s::jsonb)
   ```
   The surrounding code (`reasons/pg.py:3263`) documents the intent: "Use JSONB containment: antecedents @> '[\"node_id\"]'".

3. **The operator is index-compatible.** PostgreSQL's default JSONB GIN opclass (`jsonb_ops`) supports `@>` (containment), so both indexes *can* service the two sides of the OR. No `jsonb_path_ops` is specified, which is fine — `jsonb_ops` is actually *required* for plain `@>`.

### What we *cannot* conclude from the code alone

- **No `EXPLAIN`/`ANALYZE` plumbing exists.** `find_query_execution_helpers` returned **0 matches**, so there is no existing benchmark, plan-capture, or profiling code in the repo. You can't point to a test or log that shows the plan.
- **The full query is truncated.** We only see the OR clause (line 3270) and a comment (line 3263). We don't see:
  - whether there is a `domain_id = %s` predicate alongside it (the table has a `domain_id UUID NOT NULL` column),
  - whether `node_id` is also filtered,
  - what the `WHERE`/`JOIN` context is,
  - whether it's part of `_compute_truth` or a standalone lookup.
- **No table size or statistics** are given. Planner choice depends heavily on `pg_class.reltuples`, `pg_statistic`, and selectivity estimates.

### Why this is a genuinely uncertain question

For an `index_a @> X OR index_b @> X` predicate, PostgreSQL has *several* plausible plans:

1. **BitmapOr of two GIN Bitmap Index Scans** — the "ideal" plan. The planner will do this when both sides are cheaply estimated and the combined selectivity favors it.
2. **Sequential scan** — if the table is small or the OR is estimated to be non-selective, PostgreSQL will *deliberately* ignore both GIN indexes. This is common for small `rms_justifications` tables and directly contradicts "uses both GIN indexes."
3. **One index + filter** — if one side is highly selective and the other isn't, the planner may scan one index and filter the OR (or use the other predicate if present).
4. **Different index entirely** — if there's a `domain_id` filter, the `idx_rms_justifications_node (node_id, domain_id)` index might dominate.

The specific behavior depends on:
- Table cardinality (`rms_justifications` row count per domain).
- Selectivity of the two `@>` predicates.
- `effective_cache_size`, `random_page_cost`, `work_mem` (relevant to bitmap heap cost).
- Whether the query is inside a larger transaction/CTE that changes cost modeling.
- Whether `ANALYZE` has been run since index creation.

None of that is in the observations.

### Recommendation

Yes, benchmark. Concretely:

- Capture `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` for the full query at realistic cardinality (e.g., 100k–1M justifications across multiple domains).
- Compare against: (a) splitting the OR into a `UNION` of two queries, each hitting one index, and (b) a `UNION ALL` with `DISTINCT`. This is a standard trick when the OR plan degenerates.
- Check `pg_stat_user_indexes.idx_scan` for both `idx_rms_justifications_antecedents` and `idx_rms_justifications_outlist` before/after a workload to confirm both indexes actually get hit.
- Consider whether the two columns should be combined into a single GIN index `USING gin((antecedents || outlist))` if the OR pattern is dominant.

### What's missing from the observations

- The complete SQL around `reasons/pg.py:3260–3280` (the surrounding `WHERE`, `SELECT`, and any CTE/`JOIN`).
- Any `EXPLAIN` output or `pg_stat_user_indexes` data.
- Row counts / `reltuples` for `rms_justifications`.
- The name of the enclosing function (only the line is given, not the def).
- A test or benchmark harness — `find_query_execution_helpers` was empty, so none exists yet.

## Topics to Explore

- [file] `reasons/pg.py` — Read the full SQL block around lines 3260–3285 to see whether other predicates (e.g., `domain_id`, `node_id`) are present and might dominate the plan.
- [general] `EXPLAIN-ANALYZE-benchmark` — Add a benchmark script that runs `EXPLAIN (ANALYZE, BUFFERS)` against realistic row counts for `rms_justifications` to confirm whether BitmapOr of both GIN indexes is chosen.
- [general] `index-usage-counters` — Query `pg_stat_user_indexes` for `idx_rms_justifications_antecedents` and `idx_rms_justifications_outlist` to verify both are actually scanned at runtime.
- [function] `reasons/pg.py:_compute_truth` — Understand whether this OR query is on the hot path of every truth-value recomputation, which would amplify any plan regression.
- [general] `jsonb-ops-vs-jsonb-path-ops` — Compare the default `jsonb_ops` (supports `@>`) against `jsonb_path_ops` (smaller, but only supports `@>`); the code currently relies on the default.

## Beliefs

- `antecedents-and-outlist-have-own-gin-indexes` — Both `idx_rms_justifications_antecedents` and `idx_rms_justifications_outlist` are created as GIN indexes on their respective JSONB columns (`reasons/pg.py:85-86`).
- `jsonb-containment-is-supported-by-default-gin-opclass` — The `@>` operator used by the query is compatible with PostgreSQL's default `jsonb_ops` GIN opclass, so both indexes are *eligible* for the OR predicate.
- `or-predicate-plan-is-not-determinable-from-code` — Whether the planner picks BitmapOr (both indexes) vs. a sequential scan vs. a single-index plan depends on runtime statistics not present in the observations.
- `no-explain-tooling-exists-in-codebase` — No `EXPLAIN`/`ANALYZE`/`query_plan` helpers were found (`find_query_execution_helpers.total_count == 0`), so there is no existing benchmark or plan-inspection utility.
- `query-context-is-partially-visible` — Only the `AND (antecedents @> ... OR outlist @> ...)` fragment at `reasons/pg.py:3270` is captured; the enclosing `WHERE`, `SELECT`, and any `domain_id`/`node_id` filters are missing.
