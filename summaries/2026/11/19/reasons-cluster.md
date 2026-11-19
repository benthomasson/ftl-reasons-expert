---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/cluster.py

# `reasons/cluster.py`

## Purpose

`reasons/cluster.py` owns semantic clustering and budget-aware belief selection for the derive pipeline. It takes a `{node_id: text}` mapping of beliefs, embeds the texts with `sentence-transformers`, clusters them with KMeans, and returns selected belief IDs under a budget.

It supports three public behaviors:

- `cluster_beliefs`: sample across cluster boundaries for cross-domain coverage.
- `cluster_beliefs_intra`: focus one cluster per derive round for topically adjacent combinations.
- `list_clusters`: expose full cluster assignments for API/CLI inspection.

It does not persist beliefs, mutate storage, or call LLMs. The optional clustering dependencies are guarded so the module can be imported even when the `[cluster]` extra is not installed.

## Key Components

### Optional dependency guard

- `HAS_CLUSTER_DEPS` is set by a single `try/except ImportError` around `SentenceTransformer`, `numpy`, and `sklearn.cluster.KMeans`.
- If any import fails, all three names are set to `None` and `HAS_CLUSTER_DEPS = False`. The module remains importable.
- `_require_cluster_deps()` raises an `ImportError` with the install hint: `pip install 'ftl-reasons[cluster]'`.
- `DEFAULT_MODEL = "all-MiniLM-L6-v2"`.

### `ClusterCache`

`ClusterCache(model_name=DEFAULT_MODEL)`:

- Calls `_require_cluster_deps()` immediately.
- Instantiates `SentenceTransformer(model_name)`.
- Stores `model_name` and an internal `_cache` dict.

`embed(beliefs)`:

- Accepts `{node_id: text}`.
- Sorts node IDs so embeddings are returned in deterministic order.
- Cache key is `(node_id, sha256(text.encode()).hexdigest()[:16])`.
- Only uncached texts are passed to `model.encode(..., show_progress_bar=False)`.
- Returns `(ids, np.array(embeddings))`, where `ids` is sorted and embeddings match that order.

Important detail: cache entries are keyed by both node ID and text hash, so changed text invalidates that node’s cached embedding. Same text under different node IDs is cached separately.

### `_auto_k(n_beliefs, n_clusters=None, max_k=20)`

Computes cluster count:

- If `n_clusters` is provided, `k = max(n_clusters, 1)`.
- Otherwise `k = n_beliefs // 5`, then `min(k, max_k)`, then `max(k, 2)`.
- Finally returns `min(k, n_beliefs)`.

So explicit `n_clusters` is clamped to at least 1 and at most `n_beliefs`. Auto mode effectively has a floor of 2 for non-empty inputs, even if `max_k < 2`, because `max(..., 2)` happens after `min(..., max_k)`.

### `cluster_beliefs(...)`

Signature:

```python
cluster_beliefs(beliefs, budget, seed=None, n_clusters=None,
                cache=None, model_name=DEFAULT_MODEL)
```

Behavior:

- Requires cluster deps before the fast path.
- If `len(beliefs) <= budget`, returns all belief IDs immediately with stats `n_clusters=1`, `cluster_sizes=[len(beliefs)]`, `embedding_model=model_name`.
- Otherwise creates a `ClusterCache` if no cache was supplied, embeds beliefs, computes `k = _auto_k(len(beliefs), n_clusters, max_k=min(budget // 3, 20))`, and runs `KMeans(..., random_state=seed, n_init=10)`.
- Groups IDs by KMeans label.
- Allocates budget across clusters:
  - `base_per = budget // k`
  - `remainder = budget % k`
  - Clusters are sorted by descending size.
  - The first `remainder` clusters get one extra allocation.
  - Each allocation is capped at cluster size.
  - Samples within each cluster using `random.Random(seed)`.
- Returns `(selected_ids, stats)` with `n_clusters`, `cluster_sizes`, and `embedding_model`.

`cluster_beliefs` does not redistribute allocation when a cluster is smaller than its allocated share, so it can select fewer than `budget` IDs.

### `cluster_beliefs_intra(...)`

Signature:

```python
cluster_beliefs_intra(beliefs, budget, round_num=0, seed=None,
                      n_clusters=None, cache=None,
                      model_name=DEFAULT_MODEL)
```

Behavior:

- Same dependency check, fast path, embedding, auto-k, and KMeans setup as `cluster_beliefs`.
- Clusters are sorted by descending size.
- Focus cluster index is `round_num % k`.
- The focused cluster is `sorted_labels[focus_idx]`.
- Samples up to `min(budget, len(members))` IDs from that one cluster.
- Returns stats including `focus_cluster: focus_idx`.

Important: `focus_cluster` is an index into the size-sorted cluster list, not the original KMeans label. The actual focused label may differ.

### `list_clusters(...)`

Signature:

```python
list_clusters(beliefs, n_clusters=None, seed=None, cache=None,
              model_name=DEFAULT_MODEL)
```

Behavior:

- Requires cluster deps first.
- If `len(beliefs) <= 3`, returns one synthetic cluster with `id: 0` and beliefs sorted by node ID.
- Otherwise embeds, computes auto-k with default `max_k=20`, runs KMeans, and groups full `{"id": nid, "text": text}` records by KMeans label.
- Returns clusters sorted by descending member count.
- Cluster IDs are the original KMeans labels, not reindexed.
- Returns `{"clusters": ..., "n_clusters": k, "embedding_model": ...}`.

## Patterns

- **Optional dependency pattern**: guarded imports plus `_require_cluster_deps()` allow the rest of `reasons` to import this module without the `[cluster]` extra.
- **Lazy model loading**: no model is created at import time. `ClusterCache` creates the model, and public functions create a cache only when actual clustering is needed.
- **Embedding cache**: `ClusterCache` supports `--exhaust`-style reuse across derive rounds. The cache key includes text hash for invalidation.
- **Deterministic sampling**: KMeans uses `random_state=seed`; sampling uses an isolated `random.Random(seed)`.
- **Stratified-ish cross-cluster sampling**: `cluster_beliefs` allocates budget across clusters, with extra slots going to largest clusters.
- **Round-robin intra-cluster focus**: `cluster_beliefs_intra` rotates through clusters by `round_num % k`.
- **Stats-rich return values**: selected IDs are returned alongside cluster metadata for logging/observability.

## Dependencies

Standard library:

- `random`
- `hashlib.sha256`

Optional third-party:

- `sentence_transformers.SentenceTransformer`
- `numpy`
- `sklearn.cluster.KMeans`

Imported by:

- `reasons/api.py`
- `reasons/cli.py`
- `reasons/contradictions.py`
- `reasons/derive.py`
- `reasons_lib/cluster.py`
- tests including `tests/test_api.py`, `tests/test_cli.py`, `tests/test_cluster.py`, `tests/test_contradictions.py`, `tests/test_derive.py`

The `reasons_lib` namespace also references this module per the import graph, so changes here affect that package surface as well.

## Flow

For `cluster_beliefs` and `cluster_beliefs_intra`:

1. Call `_require_cluster_deps()`.
2. If `len(beliefs) <= budget`, return all IDs and simple stats.
3. Reuse supplied `cache` or create `ClusterCache(model_name)`.
4. Embed beliefs into sorted IDs plus a NumPy embedding matrix.
5. Compute `k` via `_auto_k`.
6. Fit KMeans and get labels.
7. Group IDs into `clusters: {label: [node_id, ...]}`.
8. Build `cluster_sizes` in numeric label order.
9. Select IDs:
   - `cluster_beliefs`: allocate across size-sorted clusters and sample each.
   - `cluster_beliefs_intra`: pick one size-sorted cluster by round and sample it.
10. Return selected IDs and stats.

For `list_clusters`:

1. Call `_require_cluster_deps()`.
2. If `len(beliefs) <= 3`, return one cluster sorted by ID.
3. Otherwise embed, auto-k, KMeans, group full records by label, sort clusters by size, and return all assignments.

## Invariants

- Every public clustering function calls `_require_cluster_deps()` before any fast path. Missing optional dependencies raise `ImportError` even when the function would have skipped clustering.
- `_auto_k` clamps cluster count to `min(k, n_beliefs)` and explicit `n_clusters` to at least 1.
- Embedding path is deterministic: `ClusterCache.embed` sorts node IDs and returns embeddings in that order.
- `cluster_beliefs` and `cluster_beliefs_intra` fast paths return `list(beliefs.keys())` without sorting; `list_clusters` fast path sorts items by node ID.
- Selected IDs are subsets of input IDs and are not duplicated because cluster groups are disjoint and sampling is within disjoint groups.
- Selected count is at most `budget`, but not guaranteed to equal `budget`; small clusters are capped by size and not refilled.
- `focus_cluster` in `cluster_beliefs_intra` is an index into the descending-size-sorted cluster list, not a KMeans label.
- `list_clusters` returns original KMeans label IDs, sorted by descending cluster size.
- Fast-path stats use the `model_name` argument; clustered paths use `cache.model_name` when a cache is used.

## Error Handling

- Missing optional dependencies produce a targeted `ImportError` from `_require_cluster_deps()` with the `[cluster]` install instruction.
- The module import itself never fails due to missing `sentence-transformers`, `numpy`, or `scikit-learn`.
- Errors from model loading, text encoding, KMeans fitting, or invalid model names are not caught and propagate to the caller.
- There is no fallback clustering or non-cluster sampling if the optional dependencies are missing.
- Inputs are not validated for negative budgets, non-string belief text, or empty/edge-case cluster counts.
- KMeans behavior with empty or duplicate clusters is not explicitly handled. `cluster_beliefs_intra` uses `round_num % k` and indexes the observed sorted labels, so a KMeans run returning fewer distinct labels than requested `k` could cause selection issues.

## Topics to Explore

- [file] `reasons/derive.py` — How `cluster_beliefs` and `cluster_beliefs_intra` are wired into derive rounds and cache reuse.
- [function] `reasons/cluster.py:ClusterCache.embed` — Embedding cache keying, invalidation, and reuse semantics.
- [file] `tests/test_cluster.py` — Expected behavior for auto-k, budget allocation, fast paths, and intra-cluster rotation.
- [file] `reasons/api.py` — How clustering results are exposed through API endpoints or service methods.
- [general] `optional-dependency-extras` — How `ftl-reasons[cluster]` is declared in `pyproject.toml` and guarded across the project.

## Beliefs

- `cluster-public-functions-require-cluster-extras` — Every public clustering function calls `_require_cluster_deps()` before its fast path, so missing sentence-transformers/scikit-learn raises `ImportError` even when clustering would be skipped.
- `cluster-cache-keys-on-node-id-and-text-hash` — `ClusterCache.embed` invalidates cached embeddings when belief text changes by keying entries on `(node_id, sha256(text)[:16])`.
- `cluster-inter-sampling-is-not-budget-guaranteed` — `cluster_beliefs` can return fewer than `budget` IDs because per-cluster allocations are capped by cluster size and not redistributed.
- `cluster-intra-rotates-size-sorted-clusters` — `cluster_beliefs_intra` selects the focus cluster by `round_num % k` after sorting clusters by descending size.
- `cluster-auto-k-caps-by-budget-thirds` — `cluster_beliefs` and `cluster_beliefs_intra` use `max_k=min(budget // 3, 20)` for auto-k, while `list_clusters` uses the default `max_k=20`.
