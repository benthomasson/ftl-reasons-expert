---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: cluster_beliefs in reasons/cluster.py

## Purpose

`cluster_beliefs` selects a subset of belief node IDs by semantically clustering belief texts and then sampling a quota from each cluster. It exists to support cross-domain/topical belief selection in the derive pipeline: instead of taking an arbitrary prefix or purely random sample, it tries to cover multiple semantic clusters while staying within `budget`.

If all beliefs already fit in `budget`, it skips embedding and clustering entirely and returns all belief IDs.

---

## Contract

### Preconditions

- The optional clustering dependencies must be installed: `sentence-transformers`, `numpy`, and `scikit-learn`.
- `beliefs` is expected to be a mapping like `{node_id: text}`.
- `budget` is expected to be a non-negative integer.
- `cache`, if provided, is expected to be a `ClusterCache`-compatible object with `embed(...)` and `model_name`.
- If clustering is actually performed, belief IDs must be mutually sortable because `ClusterCache.embed` does `sorted(beliefs.keys())`.
- If clustering is actually performed, belief texts must be strings because `ClusterCache.embed` calls `text.encode()` for hashing.

### Postconditions

- Returns a tuple `(selected_ids, cluster_stats)`.
- `selected_ids` is a list of unique node IDs drawn from `beliefs`.
- `len(selected_ids) <= budget`, but it may be strictly less than `budget` because per-cluster quotas are capped by cluster size and unused quota is not redistributed.
- Does not mutate the `beliefs` mapping.
- May mutate a provided `ClusterCache` by filling its embedding cache.
- Uses a local `random.Random(seed)` for sampling; it does not touch Python’s global RNG.
- In the shortcut path, `selected_ids` preserves `beliefs.keys()` insertion order.
- In the clustered path, `selected_ids` is ordered by cluster iteration order, i.e. clusters sorted by descending size, then sampled members within each cluster.

### Key invariants

- `k` is clamped to at least `1` and at most `len(beliefs)` when passed explicitly, and is at least `2` for auto mode when `len(beliefs) >= 2`.
- `cluster_stats["cluster_sizes"]` is ordered by ascending KMeans label order, not by cluster size.
- The sampling loop iterates clusters in descending size order, so remainder quota is biased toward larger clusters.
- The function does not guarantee exactly `budget` selected beliefs, even when there are enough beliefs.

---

## Parameters

| Parameter | Meaning | Expected type/value | Edge cases |
|---|---|---|---|
| `beliefs` | Mapping of belief node ID to belief text | `dict[str, str]` in normal use | If `len(beliefs) <= budget`, texts are not validated and IDs are not sorted. If clustering runs, IDs must be sortable and texts must be strings. |
| `budget` | Maximum number of beliefs to select | Non-negative `int` | `0` returns an empty selection in the clustered path. Negative or non-integer values are not validated and can cause `ValueError`/`TypeError` later. |
| `seed` | Random seed for KMeans and Python sampling | `int` or `None` | Passed to `KMeans(random_state=seed)` and `random.Random(seed)`. `None` makes sampling non-deterministic. |
| `n_clusters` | Explicit cluster-count override | `int` or `None` | `None` uses `_auto_k`. If not `None`, `_auto_k` clamps to at least `1` and at most `len(beliefs)`. Values like `0` or negative become `1`. |
| `cache` | Optional embedding cache | `ClusterCache` or `None` | If `None`, a new `ClusterCache(model_name)` is created, loading the sentence-transformer model. If provided, it is reused and mutated. |
| `model_name` | Sentence-transformer model name | `str` | Used when constructing a new cache, and in shortcut-path stats. If a `cache` is provided, the clustered path reports `cache.model_name`, not this argument. |

---

## Return Value

Returns:

```python
(selected_ids, cluster_stats)
```

### Shortcut path: `len(beliefs) <= budget`

Returns:

```python
list(beliefs.keys()), {
    "n_clusters": 1,
    "cluster_sizes": [len(beliefs)],
    "embedding_model": model_name,
}
```

Notes:

- No embeddings are computed.
- `n_clusters` is reported as `1` even if `beliefs` is empty.
- `cluster_sizes` is `[0]` for an empty `beliefs` mapping.
- `embedding_model` is the `model_name` argument, not `cache.model_name`, even if a `cache` was passed.

### Clustered path: `len(beliefs) > budget`

Returns:

```python
selected, {
    "n_clusters": k,
    "cluster_sizes": cluster_sizes,
    "embedding_model": cache.model_name,
}
```

Caller obligations:

- Do not assume `len(selected) == budget`.
- Do not assume selected IDs are in original belief order.
- Do not assume `cluster_sizes` is sorted by size; it is sorted by KMeans label.
- Handle `selected == []` when `budget == 0` or when quotas sample nothing.

---

## Algorithm

1. **Require optional dependencies**

   Calls `_require_cluster_deps()`. If `sentence-transformers`, `numpy`, or `scikit-learn` are missing, raises `ImportError`.

   This happens before the shortcut check, so even `len(beliefs) <= budget` still requires the optional cluster dependencies to be installed.

2. **Shortcut when all beliefs fit**

   If `len(beliefs) <= budget`, returns all belief keys and synthetic stats with one cluster.

3. **Create or reuse embedding cache**

   If `cache is None`, constructs `ClusterCache(model_name)`, which loads the sentence-transformer model.

4. **Embed beliefs**

   Calls:

   ```python
   ids, embeddings = cache.embed(beliefs)
   ```

   `ClusterCache.embed`:

   - Sorts node IDs.
   - Reuses cached embeddings keyed by `(node_id, sha256(text)[:16])`.
   - Embeds only uncached or changed texts.
   - Returns `ids` in sorted order and embeddings in matching order.

5. **Choose cluster count**

   Calls:

   ```python
   k = _auto_k(len(beliefs), n_clusters, max_k=min(budget // 3, 20))
   ```

   `_auto_k` behavior:

   - If `n_clusters` is explicitly provided:
     ```python
     k = max(n_clusters, 1)
     return min(k, n_beliefs)
     ```
     The `max_k` argument is ignored.

   - Otherwise:
     ```python
     k = n_beliefs // 5
     k = min(k, max_k)
     k = max(k, 2)
     return min(k, n_beliefs)
     ```

   So default `k` is roughly `n_beliefs / 5`, clamped to at least `2` when there are at least two beliefs, and normally capped by `max_k`. Because the `max(k, 2)` happens after `min(k, max_k)`, the default `k` can exceed `max_k` when `max_k < 2`.

6. **Run KMeans**

   ```python
   km = KMeans(n_clusters=k, random_state=seed, n_init=10)
   labels = km.fit_predict(embeddings)
   ```

   Uses Euclidean KMeans over sentence embeddings.

7. **Group IDs by cluster label**

   Iterates over `ids` and `labels`, building:

   ```python
   clusters[label] = [node_id, ...]
   ```

   Because `ids` came from `cache.embed`, members within each cluster are ordered by sorted node ID.

8. **Compute cluster stats**

   ```python
   cluster_sizes = [len(clusters[c]) for c in sorted(clusters)]
   ```

   This orders sizes by ascending KMeans label, not by size. If all labels `0..k-1` are present, this list has length `k` and sums to `len(beliefs)`.

9. **Compute sampling quotas**

   ```python
   rng = random.Random(seed)
   base_per = budget // k
   remainder = budget % k
   ```

   Conceptually, each cluster gets `base_per` selections, and the first `remainder` clusters in the iteration order get one extra.

10. **Sort clusters by descending size**

    ```python
    sorted_labels = sorted(clusters, key=lambda c: -len(clusters[c]))
    ```

    Larger clusters are processed first. Ties preserve the insertion order of `clusters`, which comes from the order labels are first encountered in the sorted `ids` list.

11. **Sample from each cluster**

    For each sorted cluster `i`:

    ```python
    alloc = base_per + (1 if i < remainder else 0)
    alloc = min(alloc, len(members))
    selected.extend(rng.sample(members, alloc))
    ```

    `rng.sample` samples without replacement within each cluster. Since clusters are disjoint, selected IDs are unique.

    Important: if a cluster has fewer members than its quota, the quota is capped and the unused selection is not redistributed to other clusters. Therefore the final selected count can be less than `budget`.

12. **Return selection and stats**

    Returns `selected` plus `n_clusters`, `cluster_sizes`, and `cache.model_name`.

---

## Side Effects

- Creates a `ClusterCache` if none was provided.
- Loading a `ClusterCache` constructs a `SentenceTransformer`, which may read model files from disk and may download model files if not cached locally.
- If a cache is provided, `cache.embed(...)` mutates `cache._cache` by storing newly computed embeddings.
- Runs KMeans, which performs CPU work and may use multiple threads depending on the environment.
- Uses a local `random.Random` instance; it does not mutate the global `random` module state.
- Does not mutate `beliefs`.
- Does not write files or return errors via special return values.

---

## Error Handling

Errors are not swallowed; they propagate to the caller.

Likely exceptions:

- `ImportError` from `_require_cluster_deps()` if optional clustering dependencies are missing.
- Errors from `SentenceTransformer(model_name)` such as model download/load failures.
- `TypeError` or similar if belief IDs are not mutually sortable during `sorted(beliefs.keys())`.
- `AttributeError` if belief texts are not strings and `cache.embed` attempts `text.encode()`.
- `ValueError` from `KMeans` if cluster count or embedding shape is invalid.
- `ValueError` from `random.Random.sample` if a negative allocation is produced, which can happen with invalid negative `budget`.
- `TypeError` if `budget` is not an integer in contexts where integer division/sample sizes are expected.

The function assumes `budget` is valid and non-negative but does not enforce it.

---

## Usage Patterns

Typical usage:

```python
selected_ids, stats = cluster_beliefs(
    beliefs=current_beliefs,
    budget=remaining_budget,
    seed=42,
    cache=shared_cache,
)
```

Common patterns:

- Called from the derive pipeline when cluster mode is enabled.
- A shared `ClusterCache` may be passed across multiple derive rounds to avoid re-embedding unchanged beliefs.
- `n_clusters` can be supplied for deterministic tests or user-controlled clustering.
- `seed` is used for reproducible KMeans and sampling when embeddings are deterministic.
- Callers should not assume exactly `budget` beliefs are returned.
- Callers should not rely on the returned selection order for semantic meaning.
- If passing a cache, callers should ensure the cache model matches their intent. The `model_name` argument is not used for embedding when a cache is provided.

---

## Dependencies

Internal dependencies:

- `_require_cluster_deps()` — validates optional dependencies.
- `ClusterCache` — embeds and caches belief texts.
- `_auto_k()` — computes cluster count.
- `DEFAULT_MODEL` — default sentence-transformer model name.

External dependencies:

- `sentence_transformers.SentenceTransformer`
- `numpy`
- `sklearn.cluster.KMeans`
- `random`
- `hashlib.sha256` inside `ClusterCache`

Optional installation is expected via:

```bash
pip install 'ftl-reasons[cluster]'
```

---

## Assumptions Not Enforced by Types

- `budget` is assumed to be a non-negative integer, but this is not validated.
- `beliefs` is assumed to be a mapping, but only `len`, `list(beliefs.keys())`, item lookup, and `.keys()` are used.
- If clustering runs, belief IDs must be sortable. Mixed-type IDs can raise during `sorted(beliefs.keys())`.
- If clustering runs, belief texts must be strings because they are hashed via `text.encode()`.
- A provided `cache` must expose `embed(...)` and `model_name`.
- The code implicitly expects KMeans to produce all labels `0..k-1`; `cluster_sizes` is built only from labels actually present in `clusters`.
- The `model_name` argument and a provided `cache.model_name` are not checked for consistency.
- In the shortcut path, `embedding_model` reports the `model_name` argument even if a different-model `cache` was passed.

---

## Topics to Explore

- [function] `reasons/cluster.py:ClusterCache.embed` — Explains how embeddings are sorted, cached, and reused across derive rounds.
- [function] `reasons/cluster.py:_auto_k` — Details the exact cluster-count clamping behavior and its interaction with `budget`.
- [file] `tests/test_cluster.py` — Shows expected behavior for budgeting, clustering, and edge cases.
- [general] `derive-pipeline-cluster-mode` — Explains how `cluster_beliefs` and `cluster_beliefs_intra` are used by the derive pipeline.
- [general] `cluster-budget-underfill` — Investigates cases where quota caps cause fewer than `budget` beliefs to be selected.

## Beliefs

- `cluster-beliefs-requires-cluster-deps-even-when-short-circuiting` — Calls `_require_cluster_deps()` before checking `len(beliefs) <= budget`, so missing optional dependencies raise `ImportError` even when no embedding is needed.
- `cluster-beliefs-returns-all-keys-in-insertion-order-when-within-budget` — The shortcut returns `list(beliefs.keys())` with stats using `n_clusters=1`, `cluster_sizes=[len(beliefs)]`, and the `model_name` argument.
- `cluster-beliefs-may-return-fewer-than-budget` — Per-cluster quotas are capped by cluster size and unused quota is not redistributed, so `len(selected_ids)` can be less than `budget`.
- `cluster-beliefs-mutates-provided-cache` — When a `ClusterCache` is passed, `cache.embed` adds new `(node_id, text_hash)` embeddings to its internal cache.
- `cluster-beliefs-cluster-sizes-use-label-order` — `cluster_stats["cluster_sizes"]` is ordered by ascending KMeans label, while sampling iterates clusters in descending size order.
