---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: cluster_beliefs_intra in reasons/cluster.py

## `cluster_beliefs_intra`

### Purpose
`cluster_beliefs_intra` implements an **intra-cluster** belief-selection strategy. It embeds belief texts, clusters them semantically with KMeans, then spends the round’s selection budget on **one focus cluster** rather than spreading selections across all clusters.

It exists to support derive rounds where the LLM should receive topically adjacent beliefs that are more likely to combine productively. Across repeated rounds, `round_num` rotates the focus cluster so different semantic groups are eventually explored.

This is the contrasting strategy to `cluster_beliefs`, which samples across cluster boundaries.

### Contract
Preconditions:
- Optional clustering dependencies must be installed: `sentence-transformers` and `scikit-learn`. This is checked **before** the early-return path, so even when no clustering is needed, missing deps still raise.
- `beliefs` is expected to be a mapping of node IDs to text, typically `dict[str, str]`.
- Node IDs must be sortable because `ClusterCache.embed` does `sorted(beliefs.keys())`.
- Belief values must be strings because the cache hashes `text.encode()`.
- `budget` is expected to be a non-negative integer.
- `round_num`, `seed`, and `n_clusters` are expected to be integers/`None` where appropriate.
- If `cache` is supplied, it should be a compatible `ClusterCache` instance, ideally created with the same embedding model.

Postconditions:
- Returns `(selected_ids, cluster_stats)`.
- If `len(beliefs) <= budget`, returns **all** belief IDs and stats with `n_clusters=1`, `cluster_sizes=[len(beliefs)]`, `focus_cluster=0`, and `embedding_model=model_name`.
- Otherwise, returns a random sample from exactly one focus cluster.
- `selected_ids` length is at most `budget`, and may be shorter if the focus cluster has fewer than `budget` members.
- `selected_ids` contains unique node IDs from `beliefs`.
- `beliefs` is not mutated.

Important invariant/ambiguity:
- `cluster_sizes` is built in **ascending cluster-label order** via `sorted(clusters)`.
- `focus_cluster` is an index into clusters sorted by **descending size**.
- Therefore `cluster_sizes[focus_cluster]` is **not guaranteed** to be the size of the focused cluster.

### Parameters
- `beliefs`: Mapping from node ID to belief text. Empty input returns `[]` with one synthetic cluster when `budget >= 0`.
- `budget`: Maximum number of beliefs to select. If `budget >= len(beliefs)`, the function returns all beliefs immediately. If `budget == 0` and beliefs are non-empty, it still clusters and returns an empty selection.
- `round_num`: Used as `round_num % k` to choose which size-sorted cluster to focus on. Intended to be incremented across derive rounds. Negative values technically work via Python modulo but are not expected.
- `seed`: Passed to `KMeans(random_state=seed)` and to `random.Random(seed)`. Controls clustering and sampling reproducibility. `None` makes behavior nondeterministic.
- `n_clusters`: Optional override for cluster count. It is coerced to at least `1` and capped at `len(beliefs)`. When provided, the budget-derived `max_k` cap is ignored.
- `cache`: Optional `ClusterCache`. If omitted, a new one is created with `model_name`. If supplied, its embeddings and `model_name` are used for the clustered path.
- `model_name`: Sentence-transformers model name. Used when creating a cache and in the early-return stats. If a `cache` is supplied, the actual embeddings use `cache.model_name`; `model_name` is not used for embedding.

### Return Value
Returns a tuple:

```python
(selected_ids, cluster_stats)
```

`selected_ids` is a list of belief node IDs:
- In the early-return path, it is `list(beliefs.keys())`, preserving dict insertion order.
- In the clustered path, it is `rng.sample(members, alloc)`, so order is randomized and length is `alloc <= budget`.

`cluster_stats` is a dict with:
- `n_clusters`: The `k` passed to KMeans, or `1` in the early-return path.
- `cluster_sizes`: List of cluster sizes in ascending cluster-label order.
- `embedding_model`: In the clustered path, `cache.model_name`; in the early-return path, the `model_name` parameter.
- `focus_cluster`: Index into clusters sorted by descending size, in `[0, k-1]` normally. In the early-return path, `0`.

Caller obligations:
- Do not assume `len(selected_ids) == budget`.
- Do not assume `focus_cluster` indexes into `cluster_sizes`.
- Handle the early-return stats shape where `n_clusters == 1`.
- If using a custom cache, ensure its model matches the reported/stated model expectation.

### Algorithm
1. Call `_require_cluster_deps()` immediately. This raises `ImportError` if optional clustering dependencies are unavailable.
2. If `len(beliefs) <= budget`, return all belief IDs and synthetic single-cluster stats. No embedding or KMeans runs.
3. If `cache is None`, construct `ClusterCache(model_name)`. This loads a `SentenceTransformer`.
4. Call `cache.embed(beliefs)`, which:
   - Sorts node IDs.
   - Reuses cached embeddings keyed by `(node_id, truncated_sha256(text))`.
   - Embeds uncached texts.
   - Returns `ids` and a NumPy array of embeddings ordered consistently with `ids`.
5. Compute cluster count:
   - `max_k = min(budget // 3, 20)`.
   - Call `_auto_k(len(beliefs), n_clusters, max_k=max_k)`.
   - If `n_clusters` is provided, it overrides the budget-derived cap.
   - Otherwise `k = n_beliefs // 5`, capped by `max_k`, floored at `2`, then capped at `n_beliefs`.
6. Fit `KMeans(n_clusters=k, random_state=seed, n_init=10)` on the embeddings and predict labels.
7. Build `clusters: dict[label, list[node_id]]` by iterating over `ids` and corresponding labels.
8. Compute `cluster_sizes = [len(clusters[c]) for c in sorted(clusters)]` in ascending label order.
9. Sort cluster labels by descending member count:
   ```python
   sorted_labels = sorted(clusters, key=lambda c: -len(clusters[c]))
   ```
10. Choose focus cluster index:
    ```python
    focus_idx = round_num % k
    focus_label = sorted_labels[focus_idx]
    members = clusters[focus_label]
    ```
11. Create a local RNG: `rng = random.Random(seed)`.
12. Allocate `alloc = min(budget, len(members))`.
13. Sample `alloc` unique members from the focus cluster with `rng.sample(members, alloc)`.
14. Return `selected` and stats.

### Side Effects
- May load a sentence-transformers model when creating `ClusterCache`, which can be expensive and may download/read model files.
- Mutates `cache._cache` by inserting newly computed embeddings.
- Fits KMeans, consuming CPU and memory.
- Uses a local `random.Random(seed)` instance; it does not mutate the global `random` module state.
- Does not mutate `beliefs`.
- If a `ClusterCache` is shared across threads, cache mutation is not synchronized; thread safety is the caller’s responsibility.
- The early-return path skips embedding and clustering but still requires clustering dependencies to be installed.

### Error Handling
- `_require_cluster_deps()` raises `ImportError` with install instructions if optional dependencies are missing.
- Non-string belief values cause `AttributeError` in `ClusterCache.embed` when calling `.encode()`.
- Non-sortable or mixed-type node IDs cause `TypeError` in `sorted(beliefs.keys())`.
- Negative `budget` is not validated; in the clustered path it can cause `rng.sample(..., negative)` to raise `ValueError`.
- Invalid `n_clusters` types can cause KMeans to raise.
- If KMeans somehow returns fewer distinct labels than `k`, `focus_idx = round_num % k` could index past `sorted_labels`. The code assumes KMeans returns exactly `k` non-empty clusters.
- No exceptions are caught or converted to return values; errors propagate to the caller.

### Usage Patterns
Typical usage in a derive loop:

```python
cache = ClusterCache(model_name="all-MiniLM-L6-v2")
for round_num in range(num_rounds):
    selected, stats = cluster_beliefs_intra(
        beliefs,
        budget=32,
        round_num=round_num,
        seed=42,
        cache=cache,
    )
```

Callers should:
- Reuse a `ClusterCache` across rounds to avoid recomputing embeddings.
- Pass an increasing `round_num` to rotate focus clusters.
- Use a stable `seed` for reproducible clustering and sampling.
- Treat `budget` as an upper bound, not a guarantee.
- Be careful when interpreting `cluster_sizes` and `focus_cluster` together.

### Dependencies
Internal:
- `_require_cluster_deps`
- `ClusterCache`
- `_auto_k`
- `DEFAULT_MODEL`

External:
- `sentence_transformers.SentenceTransformer`
- `numpy`
- `sklearn.cluster.KMeans`
- `random`
- `hashlib`

Optional install:
```bash
pip install 'ftl-reasons[cluster]'
```

### Assumptions Not Enforced by Types
- `beliefs` is a `dict[str, str]`-like object with sortable, homogeneous keys.
- `budget` is a non-negative integer.
- `round_num` is an integer.
- `n_clusters` is an integer or `None`.
- `cache`, if provided, is a `ClusterCache`-compatible object with `embed()` and `model_name`.
- KMeans produces exactly `k` distinct non-empty clusters, so `sorted_labels[focus_idx]` is valid.
- `cluster_sizes` and `focus_cluster` are not aligned; the code does not enforce or document this clearly.
- If `cache` is provided, its model may differ from `model_name`; the early-return path reports `model_name`, while the clustered path reports `cache.model_name`.
- Cluster rotation assumes `k` is stable across rounds; if belief count, budget, or `n_clusters` changes, `round_num % k` may skip or revisit clusters unpredictably.

## Topics to Explore
- [file] `reasons/cluster.py` — Full clustering module including cross-cluster selection and cluster listing.
- [function] `reasons/cluster.py:_auto_k` — Determines cluster count and contains important small-budget edge behavior.
- [function] `reasons/cluster.py:cluster_beliefs` — Contrasts cross-cluster sampling with intra-cluster focus selection.
- [general] `derive-cluster-mode` — How `round_num`, `cache`, and clustering strategy are wired into the derive pipeline.
- [file] `tests/test_cluster.py` — Tests covering clustering behavior, stats, and edge cases.

## Beliefs
- `cluster-beliefs-intra-requires-cluster-deps` — The function raises `ImportError` before the early-return path if clustering dependencies are missing.
- `cluster-beliefs-intra-produces-at-most-budget-ids` — The selected ID list length is always at most `budget`, but may be shorter when the focus cluster is smaller.
- `focus-cluster-rotates-by-round-num-mod-k` — The focus cluster is selected as `round_num % k` over clusters sorted by descending size.
- `cluster-sizes-do-not-map-to-focus-cluster` — `cluster_sizes` is in ascending cluster-label order, while `focus_cluster` indexes size-sorted clusters.
- `cluster-cache-enables-embedding-reuse` — Reusing a `ClusterCache` persists newly computed embeddings in its internal cache across calls.
