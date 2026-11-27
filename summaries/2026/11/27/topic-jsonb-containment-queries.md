---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Understanding PostgreSQL's `@>` semantics on JSONB arrays is essential for reasoning about correctness of `needle` matching.

## Explanation

The correctness of `needle` matching hinges on the SQL predicate around `reasons/pg.py:3263-3271`:

```python
# Use JSONB containment: antecedents @> '["node_id"]'
needle = json.dumps([nid])
...
"AND (antecedents @> %s::jsonb OR outlist @> %s::jsonb)",
(pid, needle, needle),
```

### What `@>` means for JSONB arrays

PostgreSQL’s `@>` operator means **“left JSONB value contains right JSONB value.”**

For a flat JSONB array of strings, this:

```sql
antecedents @> '["node_id"]'::jsonb
```

is true exactly when `antecedents` contains `"node_id"` as a JSON string element. It is exact JSON value equality — not substring, prefix, regex, or case-insensitive matching. For example:

```sql
'["a", "b"]'::jsonb @> '["a"]'::jsonb      -- true
'["a", "b"]'::jsonb @> '["c"]'::jsonb      -- false
'["1"]'::jsonb      @> '["1"]'::jsonb      -- true, string "1"
'[1]'::jsonb        @> '["1"]'::jsonb      -- false, number 1 vs string "1"
```

At `reasons/pg.py:3266`, `needle = json.dumps([nid])` deliberately builds a **one-element JSON array**, not a bare JSON string. That matches the column shape (`antecedents` and `outlist` are JSONB arrays) and makes the intent explicit: “does this array contain this node ID as an element?”

The comment at `reasons/pg.py:3263` documents that intent directly. The `%s::jsonb` cast at `reasons/pg.py:3270` and the parameter tuple at `reasons/pg.py:3271` pass `needle` safely as a bound parameter, avoiding SQL injection and string-built JSON.

### Why this matters for correctness

The query at `reasons/pg.py:3270` checks both sides:

```sql
antecedents @> needle OR outlist @> needle
```

So it treats a node as referenced if it appears in either the justification’s antecedents array or its outlist array. For flat string arrays, that is the correct membership test. The negative form at `reasons/pg.py:1676`:

```sql
AND NOT outlist @> %s::jsonb
```

relies on the same semantics: it excludes rows where `outlist` contains the needle. The surrounding function is not shown, so we cannot verify what `%s` is bound to there, but the operator semantics are the same.

Important caveats:

- `@>` is recursive/containment-based. If these JSONB arrays ever contain nested arrays or objects, `@>` may match more than a simple top-level element. The correctness argument here assumes the stored arrays are flat arrays of node-ID strings, which is consistent with `json.dumps(j.get("antecedents", []))` and `json.dumps(j.get("outlist", []))` in `reasons/pg.py`.
- `NOT outlist @> '[]'::jsonb` would be false for all rows because every JSONB array contains the empty array. `reasons/pg.py:3266` always builds a one-element array, so this is safe there; any other caller of the line-1676 predicate must avoid passing an empty needle.
- The schema defines `antecedents JSONB NOT NULL DEFAULT '[]'` and `outlist JSONB NOT NULL DEFAULT '[]'` in `reasons/pg.py`, so SQL NULL handling is not an issue for those columns as long as inserts preserve the non-null contract.
- The `INDEXES` block in `reasons/pg.py` creates GIN indexes on `antecedents` and `outlist`. `@>` is a GIN-supported operator, so these membership checks can use those indexes rather than unnesting arrays.

### Test coverage gap

The search in `find_jsonb_containment_tests` found zero `@>` occurrences in tests. The observed portion of `tests/test_pg.py` covers add/retract/propagation behavior, but not direct JSONB containment semantics. That means the correctness of `needle` matching currently relies on indirect integration coverage and on the assumption that the arrays remain flat string arrays.

## Topics to Explore

- [file] `reasons/pg.py` — Inspect the full enclosing functions for the `@>` usages at lines 1676 and 3263-3271 to confirm what `%s` is bound to and whether the arrays are guaranteed flat.
- [file] `tests/test_pg.py` — Add direct tests for `@>` membership: exact string match, missing string, string-vs-number distinction, nested arrays, and empty needle behavior.
- [general] `jsonb-containment-semantics` — Experiment in `psql` with `@>` on flat arrays, nested arrays, and mixed JSON types to verify the edge cases your data model permits.
- [general] `gin-index-usage` — Run `EXPLAIN ANALYZE` on the `antecedents @> ... OR outlist @> ...` query to confirm the GIN indexes from `reasons/pg.py` are used.
- [general] `node-id-json-encoding` — Verify that every node ID stored in `antecedents` and `outlist` is encoded as a JSON string via `json.dumps`, not as a number or object, because `@>` would treat those as different JSON values.

## Beliefs

- `jsonb-containment-membership-is-exact-for-flat-string-arrays` — For flat JSONB string arrays, `col @> '["id"]'::jsonb` is true exactly when `"id"` is a string element of `col`.
- `needle-is-single-element-json-array` — `reasons/pg.py:3266` builds `needle` as `json.dumps([nid])`, producing a one-element JSON array rather than a bare JSON string.
- `pg-jsonb-array-columns-are-not-null` — The `rms_justifications.antecedents` and `outlist` columns are declared `JSONB NOT NULL DEFAULT '[]'` in `reasons/pg.py`.
- `gin-indexes-support-containment-predicates` — The `INDEXES` block in `reasons/pg.py` defines GIN indexes on `antecedents` and `outlist`, enabling `@>` predicates to be indexed.
- `jsonb-containment-direct-tests-absent` — The observed search found no `@>` usage in tests, so SQL-level JSONB containment semantics are not directly covered by the observed test suite.
