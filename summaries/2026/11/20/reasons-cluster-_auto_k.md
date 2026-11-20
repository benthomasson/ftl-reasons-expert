---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: _auto_k in reasons/cluster.py

`_auto_k` is a private, pure helper that chooses the number of KMeans clusters from the number of beliefs, with support for an explicit override. It is intentionally small and has no external dependencies.

## Purpose

`_auto_k` exists to answer one question before KMeans is invoked: **how many semantic clusters should we create?** It provides a deterministic heuristic:

- If the caller passes an explicit `n_clusters`, honor it after coercing it to at least 1 and capping it at the actual belief count.
- Otherwise, derive `k` from `n_beliefs`, roughly one cluster per five beliefs, with a configured upper bound and a practical lower bound.

The “auto” heuristic is used by the clustering routines in `reasons/cluster.py` before fitting KMeans.

## Contract

For normal integer inputs and non-negative `n_beliefs`, the function postconditions are:

- If `n_clusters is not None`:
  - result is exactly `min(max(n_clusters, 1), n_beliefs)`
- If `n_clusters is None`:
  - result is exactly `min(max(min(n_beliefs // 5, max_k), 2), n_beliefs)`
- For `n_beliefs > 0`, the returned value is between `1` and `n_beliefs`.
- For `n_beliefs >= 2` in auto mode, the returned value is at least `2`.
- The result never exceeds `n_beliefs` for non-negative `n_beliefs`.
- The result is deterministic and has no side effects.

Important assumptions not enforced by the type system:

- `n_beliefs` is assumed to be a non-negative integer count. Negative values produce negative or nonsensical results.
- `n_beliefs` is assumed to match the number of embedding samples eventually passed to KMeans.
- `n_clusters`, when not `None`, is assumed to be an integer.
- `max_k` is assumed to be an integer and is only used in auto mode.
- The caller is expected to avoid passing `0` to KMeans. `_auto_k(0)` returns `0`, which is not a valid KMeans cluster count.
- `max_k` is not a strict upper bound when it is below `2`, because the lower bound of `2` is applied after `max_k`.

## Parameters

### `n_beliefs`

The number of beliefs/samples being clustered.

Expected type: `int`.

Typical source: `len(beliefs)` from the calling clustering functions.

Edge cases:

- `n_beliefs == 0` returns `0` in both explicit and auto modes because the final `min(k, n_beliefs)` caps the result at zero.
- `n_beliefs == 1` can return `1`.
- `n_beliefs >= 2` in auto mode returns at least `2`.
- Negative values are not validated and produce negative results.

### `n_clusters`

Optional explicit cluster count override.

Expected type: `None` or `int`.

Behavior:

- If not `None`, `max_k` is ignored entirely.
- Values less than `1` are silently coerced to `1`.
- Values larger than `n_beliefs` are capped by the final `min(k, n_beliefs)`.

Examples:

- `n_clusters=0`, `n_beliefs=10` → `1`
- `n_clusters=-5`, `n_beliefs=10` → `1`
- `n_clusters=100`, `n_beliefs=10` → `10`
- `n_clusters=100`, `n_beliefs=50`, `max_k=20` → `50`; `max_k` is ignored.

### `max_k`

Upper bound for the automatic heuristic only.

Expected type: `int`, default `20`.

Behavior:

- Only used when `n_clusters is None`.
- Applied before the minimum-of-2 floor.
- Therefore, `max_k=0` or `max_k=1` can still result in `2` clusters when `n_beliefs >= 2`.
- Callers often pass `max_k=min(budget // 3, 20)` in the budgeted clustering functions, so small budgets can produce `max_k` values below `2`.

## Return Value

Returns an integer cluster count intended to be passed to `KMeans(n_clusters=...)`.

The caller must handle these cases:

- For normal non-negative `n_beliefs`, the return is safe as long as it is greater than zero.
- If `n_beliefs == 0`, the return is `0`, which KMeans cannot use.
- If `n_beliefs` is negative, the return may be negative.
- If non-integer inputs are passed, the function may return non-integer values or raise later in KMeans.

## Algorithm

Step-by-step:

1. If `n_clusters is not None`:
   - Set `k = max(n_clusters, 1)`.
   - This forces explicit overrides to be at least `1`.
2. Otherwise, use the automatic heuristic:
   - Set `k = n_beliefs // 5`.
   - This targets roughly one cluster for every five beliefs.
   - Set `k = min(k, max_k)`.
   - This caps the heuristic at `max_k`.
   - Set `k = max(k, 2)`.
   - This attempts to avoid fewer than two clusters.
3. Return `min(k, n_beliefs)`.
   - This ensures the cluster count does not exceed the number of beliefs.

Examples for auto mode:

- `n_beliefs=4`, `max_k=20` → `4 // 5 = 0`; then `max(0, 2) = 2`; result `2`.
- `n_beliefs=9`, `max_k=20` → `9 // 5 = 1`; then `max(1, 2) = 2`; result `2`.
- `n_beliefs=14`, `max_k=20` → `14 // 5 = 2`; result `2`.
- `n_beliefs=15`, `max_k=20` → `15 // 5 = 3`; result `3`.
- `n_beliefs=100`, `max_k=20` → `100 // 5 = 20`; result `20`.
- `n_beliefs=100`, `max_k=5` → `100 // 5 = 20`; `min(20, 5) = 5`; result `5`.
- `n_beliefs=100`, `max_k=1` → `min(20, 1) = 1`; `max(1, 2) = 2`; result `2`.

## Side Effects

None.

`_auto_k` is a pure function:

- No mutations.
- No I/O.
- No randomness.
- No external calls.
- No access to cluster caches, models, or sklearn.

## Error Handling

The function performs no explicit validation and does not catch exceptions.

Possible issues:

- `TypeError` if inputs are not compatible with integer arithmetic or comparisons, e.g. `n_beliefs` is a string or `max_k` is `None` in auto mode.
- No `ValueError` is raised for `n_clusters < 1`; it is silently clamped to `1`.
- No error is raised for `n_beliefs == 0`; the function returns `0`.
- No error is raised for `max_k < 2`; the minimum-of-2 floor may override it.
- No error is raised for negative `n_beliefs`; the result may be negative.

Downstream KMeans can still fail if the returned value is `0`, negative, non-integer, or inconsistent with the number of embedding samples.

## Usage Patterns

`_auto_k` is called by the clustering functions in the same file:

- `cluster_beliefs`:
  ```python
  k = _auto_k(len(beliefs), n_clusters, max_k=min(budget // 3, 20))
  ```
- `cluster_beliefs_intra`:
  ```python
  k = _auto_k(len(beliefs), n_clusters, max_k=min(budget // 3, 20))
  ```
- `list_clusters`:
  ```python
  k = _auto_k(len(beliefs), n_clusters)
  ```

Caller obligations:

- Ensure `n_beliefs > 0` before passing the result to KMeans, unless the caller handles `0` separately.
- Pass integer inputs.
- Do not expect `max_k` to cap an explicit `n_clusters` override.
- Do not expect `max_k < 2` to strictly cap auto mode below `2`.

The budgeted clustering functions also use `k` for allocation:

```python
base_per = budget // k
remainder = budget % k
```

So if `k` is larger than `budget`, some clusters receive zero selections. This is tolerated by the sampling logic as long as `k > 0`.

## Dependencies

`_auto_k` itself depends only on Python built-ins.

It has no imports and does not require the optional clustering dependencies:

- No `sentence_transformers`.
- No `sklearn`.
- No `numpy`.

However, its return value is consumed by code that does use `sklearn.cluster.KMeans`.

## Topics to Explore

- [function] `reasons/cluster.py:cluster_beliefs` — Shows how `_auto_k` feeds KMeans and budget allocation, including small-budget behavior.
- [function] `reasons/cluster.py:cluster_beliefs_intra` — Uses `_auto_k` with `max_k=min(budget // 3, 20)` and rotates focus across clusters.
- [function] `reasons/cluster.py:list_clusters` — Calls `_auto_k` without a budget-derived `max_k` and guards small belief sets before clustering.
- [file] `tests/test_cluster.py` — Tests cluster-count edge cases, explicit overrides, and small belief counts.
- [general] `kmeans-cluster-count-selection` — Compare this heuristic with elbow/silhouette methods and consider why one cluster per five beliefs was chosen.

## Beliefs

- `auto-k-uses-floor-division-by-five` — In auto mode the initial candidate is `n_beliefs // 5`.
- `explicit-cluster-override-bypasses-max-k` — When `n_clusters` is not `None`, `max_k` is never read.
- `auto-k-enforces-minimum-two-for-at-least-two-beliefs` — Auto mode returns at least `2` when `n_beliefs >= 2`.
- `auto-k-applies-max-k-before-minimum-two` — Because `max_k` is applied before the floor of `2`, `max_k` values of `0` or `1` can still yield `2`.
- `auto-k-returns-zero-for-zero-beliefs` — `_auto_k(0)` returns `0`, which is not a valid KMeans cluster count.
