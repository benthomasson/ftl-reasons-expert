---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: list_clusters in reasons/cluster.py

# `list_clusters`

**Location:** `reasons/cluster.py`

## Purpose

`list_clusters` produces a **full, inspectable cluster assignment** for a set of beliefs: every input belief appears in exactly one cluster, and the complete mapping is returned to the caller. It contrasts with the other functions in the module:

- `cluster_beliefs` — clusters then *samples* `budget` beliefs across clusters.
- `cluster_beliefs_intra` — clusters then focuses on one cluster per round.
- `list_clusters` — clusters and returns **everything**, with no budget and no sampling.

It exists so a caller (CLI/API/debug tooling) can display or inspect how the semantic embedding + KMeans pipeline grouped the beliefs, including cluster sizes and which model produced the embeddings.

## Contract

### Preconditions
- Cluster extras must be installed (`sentence-transformers`, `scikit-learn`, and `numpy`). Otherwise `_require_cluster_deps()` raises `ImportError`.
- `beliefs` is a dict mapping `node_id -> text`. Node IDs must be **sortable** (used in both the shortcut path and `ClusterCache.embed`). Text values must be `str` (the cache hashes `text.encode()`).
- If a `cache` is supplied, it must already be a working `ClusterCache` (i.e., its model is loaded). The function does not validate or rebuild it.

### Postconditions
- Returns a dict with exactly three keys: `"clusters"`, `"n_clusters"`, `"embedding_model"`.
- **Conservation invariant:** sum of `len(c["beliefs"])` across all returned clusters equals `len(beliefs)` — every belief is assigned exactly once (no drops, no duplicates).
- Each `belief["text"]` comes from `beliefs[belief["id"]]`.
- Clusters are ordered by **descending member count** (`key=lambda x: -len(x[1])`), so `clusters[0]` is the largest cluster.
- Within a cluster, members are in the order returned by `cache.embed` (i.e., sorted by `node_id`), except in the small-input shortcut where they are in `sorted(beliefs.items())` order.

### Invariants / assumptions *not* enforced by the type system
- `n_clusters` is quietly coerced: `_auto_k` applies `max(n_clusters, 1)` and then `min(k, len(beliefs))`. Passing `0`, `-1`, or a value larger than `len(beliefs)` will not raise.
- When `cache` is provided, `"embedding_model"` reports `cache.model_name`, **not** the `model_name` argument. If a caller passes a cache built with a different model than `model_name`, the return value silently reflects the cache's model.
- In the shortcut path (`len(beliefs) <= 3`), `"embedding_model"` is the `model_name` argument — no embedding is actually performed.
- An **empty** `beliefs` dict satisfies `len(beliefs) <= 3` and returns one cluster with an empty `beliefs` list — arguably a quirk of the shortcut, not a documented behavior.

## Parameters

| Name | Type | Meaning | Edge cases |
|---|---|---|---|
| `beliefs` | `dict[str, str]` | `node_id -> text` to cluster | Empty dict returns one empty cluster (id 0). Node IDs must be comparable. |
| `n_clusters` | `int \| None` | Override the automatic cluster count | Values ≤ 0 are clamped to `1`; values > `len(beliefs)` are clamped to `len(beliefs)`. |
| `seed` | `int \| None` | `random_state` passed to `KMeans` | Only affects KMeans initialization; the auto-shortcut path ignores it entirely. |
| `cache` | `ClusterCache \| None` | Reuse previously computed embeddings | If `None`, a fresh `ClusterCache(model_name)` is constructed — this loads the transformer model (slow, possibly downloads weights). |
| `model_name` | `str` | Sentence-transformers model name | Only used when `cache is None` (to build the cache) **and** in the shortcut path's `"embedding_model"` field. Ignored for the returned model name if a cache is provided. |

## Return Value

```python
{
  "clusters": [
    {"id": 0, "beliefs": [{"id": "...", "text": "..."}, ...]},
    {"id": 1, "beliefs": [...]},
    ...
  ],
  "n_clusters": int,
  "embedding_model": str,
}
```

- `"clusters"` — list of cluster dicts, ordered by descending size. `"id"` is the raw KMeans label cast to `int` (typically `0..k-1`, but not guaranteed to be contiguous if sklearn's label set is sparse — in practice it is contiguous).
- `"n_clusters"` — equals `k` from `_auto_k` (or literal `1` in the shortcut path).
- `"embedding_model"` — from `cache.model_name` when a cache is used, otherwise the `model_name` argument.

The caller must handle:
- The shortcut case where `n_clusters == 1` even if more clusters were requested.
- Potentially empty `beliefs` list (only if input was empty).
- The fact that `"embedding_model"` may reflect a caller-supplied cache's model rather than `model_name`.

## Algorithm

1. **Dependency check** — `_require_cluster_deps()` raises `ImportError` with an install hint if the extras are missing.
2. **Small-input shortcut** — if `len(beliefs) <= 3`, skip embedding/clustering entirely and return a single cluster (id `0`) containing all beliefs sorted by node ID. Rationale: semantic differentiation is meaningless for ≤3 items, and this saves a model load. Note this returns `embedding_model = model_name` (the argument), not a cache's model.
3. **Ensure a cache** — if `cache is None`, instantiate `ClusterCache(model_name)`. This loads/creates a `SentenceTransformer` (potentially slow; requires network on first use).
4. **Embed** — `cache.embed(beliefs)` returns `(ids, embeddings)` where `ids` are sorted node IDs and `embeddings` is an `np.ndarray` in the same order. The cache reuses hash-keyed embeddings (`(nid, sha256(text)[:16])`) for previously seen nodes.
5. **Choose k** — `_auto_k(len(beliefs), n_clusters)`:
   - If `n_clusters` given: `k = max(n_clusters, 1)`.
   - Else: `k = min(len(beliefs) // 5, 20)`, floored at `2`.
   - Always capped by `len(beliefs)`. (Note: unlike `cluster_beliefs`, this uses the default `max_k=20` — it is **not** scaled by a budget.)
6. **Cluster** — `KMeans(n_clusters=k, random_state=seed, n_init=10).fit_predict(embeddings)`. Returns labels aligned with `ids`.
7. **Group** — iterate `ids` and append `{"id": nid, "text": beliefs[nid]}` into `groups[int(label)]`.
8. **Sort and shape** — build the output list sorted by `-len(members)` (largest cluster first) and return the result dict.

## Side Effects

- **Model loading / network I/O** — when `cache is None` and `len(beliefs) > 3`, a `SentenceTransformer` is constructed, which may download model weights the first time.
- **Cache state** — `cache.embed` populates `cache._cache` with new hash-keyed embeddings. This is a deliberate mutation enabling reuse across calls; it is the main reason the parameter exists.
- **No mutation of `beliefs`** — input dict is only read.
- **No filesystem writes or logging.**

## Error Handling

- **Raises `ImportError`** from `_require_cluster_deps()` if the cluster extras are not installed (message suggests `pip install 'ftl-reasons[cluster]'`).
- **Propagates `ValueError`/`RuntimeError`** from `sentence_transformers` (e.g., unknown `model_name`, download failure) or from `sklearn.cluster.KMeans` (e.g., degenerate configurations on pathological embeddings).
- **Does not swallow exceptions** — no `try/except` around the clustering pipeline.
- **Silent coercions** rather than validation: bad `n_clusters` values are clamped by `_auto_k`, not rejected.
- No error is returned as data — this function is not of the "capture-and-return" style.

## Usage Patterns

- Typically reached through the CLI/API layer for inspecting clusters (see `tests/test_api.py` and `tests/test_cluster.py`).
- Recommended in `--exhaust`/multi-round flows: construct **one `ClusterCache(model_name)`**, reuse it across rounds, and pass it in — otherwise each call re-instantiates the model.
- Callers that only want a bounded selection should use `cluster_beliefs` or `cluster_beliefs_intra` instead of post-processing this output.
- Callers should not assume `"embedding_model"` equals the `model_name` they passed when they also provided a `cache`.

## Dependencies

- **Internal:** `_require_cluster_deps`, `ClusterCache`, `_auto_k`, `DEFAULT_MODEL` — all in `reasons/cluster.py`.
- **External:** `scikit-learn`'s `KMeans`; indirectly `sentence_transformers.SentenceTransformer`, `numpy`, and `hashlib.sha256` (inside `ClusterCache`).
- **Import-time guarded:** the module uses a `try/except ImportError` to set `HAS_CLUSTER_DEPS`; the function re-checks this at call time.

## Topics to Explore

- [function] `reasons/cluster.py:cluster_beliefs` — the budgeted sibling that clusters then *samples*; compare its `max_k=min(budget//3, 20)` behavior with `list_clusters`'s fixed `max_k=20`.
- [function] `reasons/cluster.py:_auto_k` — the heuristic that determines `k`; verify how it clamps `n_clusters` and interacts with the `len(beliefs) <= 3` shortcut.
- [file] `reasons/cluster.py` — full module to see how `cluster_beliefs_intra` rotates clusters across rounds.
- [file] `tests/test_cluster.py` — exercises contract-level behavior of the cluster functions, including auto-`k` and edge cases.
- [file] `tests/test_api.py` — shows how `list_clusters` is surfaced through the public API/CLI.

## Beliefs

- `list-clusters-shortcut-returns-single-cluster` — When `len(beliefs) <= 3`, `list_clusters` returns one cluster with id 0 containing all beliefs and never loads a model or embeds text.
- `list-clusters-preserves-all-beliefs` — Across all returned clusters, the total number of `beliefs` entries equals `len(beliefs)` for any input.
- `list-clusters-orders-by-member-count` — The returned `clusters` list is sorted by descending cluster member count.
- `list-clusters-returns-cache-model-name` — When a `cache` is supplied, the `"embedding_model"` field reports `cache.model_name`, not the `model_name` argument.
- `list-clusters-requires-cluster-deps` — Calling `list_clusters` without `sentence-transformers`/`scikit-learn` installed raises `ImportError` before any input is processed.
