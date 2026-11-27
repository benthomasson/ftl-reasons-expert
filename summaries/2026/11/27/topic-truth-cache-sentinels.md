---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Search the repo for writes into `truth_cache` and for sentinel strings (`IN_PROGRESS`, `UNKNOWN`, `PENDING`) to pin down whether cycles block or allow justifications

## What the observations show

### `truth_cache` writes

All observed `truth_cache` activity is in `reasons/pg.py`. There are no `truth_cache` matches in `reasons/derive.py`, `reasons/network.py`, or tests.

The writes fall into two patterns:

1. **Hydration from DB rows**
   - `reasons/pg.py:3077` — `truth_cache[row[0]] = row[1]`
   - `reasons/pg.py:3154` — `truth_cache[row[0]] = row[1]`
   - `reasons/pg.py:3506` — `truth_cache = {r[0]: r[1] for r in cur.fetchall()}`
   - `reasons/pg.py:3534` — `truth_cache = {r[0]: r[1] for r in cur.fetchall()}`

2. **Updating one dependent after computing its new truth value**
   - `reasons/pg.py:3111` — `truth_cache[dep_id] = new_tv`

The cache is also repeatedly initialized as an empty local dict:
- `reasons/pg.py:3069`
- `reasons/pg.py:3146`
- `reasons/pg.py:3499`

So `truth_cache` is a **local truth-value map**: first populated from DB rows, then read while evaluating justifications, and in one path updated with `new_tv` for a dependent. It is not observed as a persistent status table or a lock table.

### How justifications read from `truth_cache`

The positive antecedent check is consistently:

```python
truth_cache.get(a, "OUT") == "IN"
```

at:
- `reasons/pg.py:3095`
- `reasons/pg.py:3157`
- `reasons/pg.py:3536`

This means: **an antecedent must be present in the cache and equal `"IN"`**. If it is missing, the default is `"OUT"`, so the antecedent check fails.

The outlist check is:

```python
truth_cache.get(o, "OUT") == "OUT"
```

at:
- `reasons/pg.py:3098`
- `reasons/pg.py:3158`
- `reasons/pg.py:3537`

This means: **an outlist node must be missing or explicitly `"OUT"`**. A missing outlist node defaults to `"OUT"`, so it does not block the justification.

There are also validation-style reads:
- `reasons/pg.py:3508` — `failed = [a for a in ants if truth_cache.get(a, "OUT") == "OUT"]`
- `reasons/pg.py:3509` — `violated = [o for o in outs if truth_cache.get(o, "OUT") == "IN"]`

These also treat missing antecedents as `"OUT"` and missing outlist nodes as `"OUT"`.

### Sentinel strings

The sentinel search found only `UNKNOWN`, not `IN_PROGRESS` or `PENDING`.

`UNKNOWN` appears in:
- `reasons/api.py:3764` — `premises_data.append({"id": pid, "text": "", "truth_value": "UNKNOWN", ...})`
- `reasons/api.py:3836` — another `"truth_value": "UNKNOWN"` fallback
- `reasons/report_belief.py:38`, `85`, `123`, `149`, `194` — `node_detail.get("truth_value", "UNKNOWN")` / `pd.get("truth_value", "UNKNOWN")`

So `UNKNOWN` is used as a **display/default fallback** in API/reporting code. It is not observed as a value written into `truth_cache` in `reasons/pg.py`.

No `IN_PROGRESS` or `PENDING` matches were found in the provided occurrences. Therefore, the observed truth-cache evaluation has **no explicit “pending” or “in progress” cycle sentinel**.

## Cycle implication from the observed code

The observed `truth_cache` logic does not contain cycle detection or a pending-state sentinel. It evaluates justifications using whatever values are currently in the cache:

```python
all(truth_cache.get(a, "OUT") == "IN" for a in ants)
```

Because missing antecedents default to `"OUT"`, an **unresolved cyclic antecedent blocks justification** unless that node already has `"IN"` in the cache.

So the direct behavior from these lines is:

- **Unknown/missing antecedent → treated as `"OUT"` → positive justification fails.**
- **Unknown/missing outlist node → treated as `"OUT"` → outlist gate is satisfied.**
- **No `IN_PROGRESS`/`PENDING` state exists in the observed `truth_cache` code.**
- **Cycles are not explicitly blocked or allowed by a sentinel.** They are evaluated against whatever DB/cache value is present, and `truth_cache[dep_id] = new_tv` at `reasons/pg.py:3111` means evaluation order can matter.

In other words, from the observed snippets, cycles are **not handled by a special “pending” state**. A cycle node can only support a justification if it is already `"IN"` in `truth_cache` at evaluation time; otherwise the default `"OUT"` blocks the antecedent check.

However, the observations do **not** include the full enclosing functions around `reasons/pg.py:3069–3111` or `reasons/pg.py:3499–3537`, so we cannot conclusively say whether the surrounding propagation loop prevents cycles, allows self-support, or reaches a fixpoint. The snippets alone are insufficient to prove the global cycle policy.

## Caveats and missing evidence

- The observations do not include the enclosing function names or full bodies for the `truth_cache` blocks in `reasons/pg.py`. Those are needed to know the iteration order and whether propagation is recursive, iterative, or snapshot-based.
- The occurrence summary says `cycle_terms_in_source.total_count == 0`, but the provided `reasons/derive.py` excerpt contains `memo[node_id] = 0  # cycle guard` inside `_get_depth`. That guard prevents infinite recursion **during depth computation**, not during truth propagation. The inconsistency suggests the search result summary may be incomplete or stale.
- `tests/test_backtracking.py` exercises chains and diamonds but no cycles.
- `tests/test_derive.py` imports and tests `_get_depth`, but no observed test covers cyclic justification truth values.
- The sentinel search found no `IN_PROGRESS` or `PENDING` anywhere in the provided matches.

## Topics to Explore

- [file] `reasons/pg.py` — Inspect the full functions enclosing lines `3069–3111` and `3499–3537` to see propagation order and whether cycles can become self-supporting.
- [file] `reasons/network.py` — Check the in-memory truth-maintenance logic for cycle detection, recursion guards, or fixpoint iteration.
- [function] `reasons/derive.py:_get_depth` — This has a visible cycle guard for depth computation; verify whether an equivalent guard exists for truth evaluation.
- [general] `cycle-propagation-tests` — No cyclic truth-justification tests were observed; add or inspect tests for self-cycles and two-node cycles.
- [general] `sentinel-truth-values` — Confirm repo-wide that `IN_PROGRESS` and `PENDING` are absent and that `UNKNOWN` never enters `truth_cache`.

## Beliefs

- `truth-cache-is-local-truth-value-map` — `reasons/pg.py` initializes `truth_cache` from DB rows and updates it with `new_tv`; it is a local truth-value cache, not a persistent status table.
- `missing-truth-cache-antecedents-default-out` — At `reasons/pg.py:3095`, `3157`, and `3536`, missing antecedents default to `"OUT"` and therefore fail `== "IN"` checks.
- `missing-truth-cache-outlist-nodes-default-out` — At `reasons/pg.py:3098`, `3158`, and `3537`, missing outlist nodes default to `"OUT"` and therefore satisfy outlist checks.
- `unknown-is-display-default-in-api-and-reporting` — `UNKNOWN` appears only in `reasons/api.py` and `reasons/report_belief.py` as a fallback value, not in the observed `truth_cache` writes.
- `derive-depth-has-cycle-guard` — `reasons/derive.py:_get_depth` sets `memo[node_id] = 0` before recursion to prevent infinite depth recursion, but this does not govern truth-justification cycles.
