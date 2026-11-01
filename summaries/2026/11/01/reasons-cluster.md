---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/cluster.py

# `reasons/cluster.py` — Semantic Clustering for Belief Sampling

## 1. Purpose

This module provides the **semantic grouping and budget-aware selection** layer for the `derive` pipeline. When the system has more beliefs than the LLM can process in one shot, `cluster.py` embeds those belief texts into vector space, groups them via KMeans, and then **selects a subset of size ≤ `budget`** from the clusters. It owns two complementary sampling strategies:

- **Cross-cluster (diverse)**: pull a few beliefs from *every* cluster so the LLM sees topically varied inputs.
- **Intra-cluster (focused)**: in each derivation round, concentrate the full budget on *one* cluster so the LLM sees topically cohesive inputs more likely to combine.

It also exposes a read-only `list_clusters` for debugging/inspection, and a `ClusterCache` so that repeated derive rounds don't re-encode the same texts.

## 2. Key Components

| Symbol | Kind | Contract |
|--------|------|----------|
| `HAS_CLUSTER_DEPS` | module-level flag | `True` only if `sentence_transformers`, `numpy`, and `sklearn` are all importable. Gates every public function. |
| `DEFAULT_MODEL` | constant | `"all-MiniLM-L6-v2"` — the default SentenceTransformer model. Small, fast, 384-dim embeddings. |
| `_require_cluster_deps()` | guard function | Raises `ImportError` with an actionable pip-install message if `HAS_CLUSTER_DEPS` is `False`. Called at the top of every public function. |
| `ClusterCache` | class | Holds a `SentenceTransformer` instance and a dict `{(node_id, text_sha256_16): ndarray}`. `embed(beliefs)` returns `(sorted_ids, np.ndarray)` in **consistent order**. New texts are encoded in one batch; cached texts are reused. |
| `_auto_k(n_beliefs, n_clusters, max_k)` | helper | Returns an integer `k` for KMeans. Formula: `min(max(n_beliefs // 5, 2), max_k)`, clamped to `[1, n_beliefs]`. Explicit `n_clusters` override skips the formula. |
| `cluster_beliefs(...)` | public API | Returns `(selected_ids: list[str], stats: dict)`. Samples **across** all clusters proportionally to cluster size. `stats` = `{n_clusters, cluster_sizes, embedding_model}`. |
| `cluster_beliefs_intra(...)` | public API | Same signature plus `round_num`. Concentrates the **entire** budget on `sorted_clusters[round_num % k]`. `stats` additionally contains `focus_cluster: int`. |
| `list_clusters(...)` | public API | Returns a full breakdown: `{"clusters": [{"id", "beliefs": [{"id","text"}]}], "n_clusters", "embedding_model"}`. No sampling — every belief is returned. |

### `ClusterCache` in detail

```python
key = (nid, sha256(text.encode()).hexdigest()[:16])
```

The cache key is a **tuple of node-id and a truncated hash of the text**. This means:
- If a belief's text changes, the old cache entry is simply unused (stale entries accumulate but are harmless).
- Two different nodes with identical text get separate cache slots (by design — they're different beliefs even if the text matches).
- The 16-hex-char truncation (64 bits) is more than enough to avoid collisions for practical belief counts.

## 3. Patterns

- **Optional dependency with graceful degradation.** The `try/except ImportError` at module level sets `HAS_CLUSTER_DEPS = False` and stubs the three symbols to `None`. This lets the rest of the package import `reasons.cluster` without crashing when the `[cluster]` extra isn't installed. Every public function calls `_require_cluster_deps()` immediately, so the error surfaces at call-site, not import-time.

- **Two sampling strategies as sibling functions** (`cluster_beliefs` vs `cluster_beliefs_intra`). They share the embed → cluster → group pipeline but diverge at the selection step. They are intentionally **not** unified behind a strategy object because the selection logic is only ~10 lines and the signatures differ (`round_num`, `focus_cluster` in stats).

- **Early-return shortcut.** All three public functions check `if len(beliefs) <= budget` (or `<= 3` for `list_clusters`) and return immediately with a single-cluster, single-group result. This avoids loading the transformer model and running KMeans when it's unnecessary.

- **Deterministic reproducibility.** `random_state=seed` on KMeans and `random.Random(seed)` for sampling mean that the same `(beliefs, budget, seed)` input always produces the same output. This is critical for the derive pipeline's backtracking and for tests.

- **`n_init=10`** on every KMeans call. This is slightly above sklearn's default of 10 (it *is* the default), but making it explicit signals intent: "we want the best of 10 restarts, not a single pass."

## 4. Dependencies

### Inbound (this file imports)
- `random` — for seeded sampling
- `hashlib.sha256` — for cache key generation
- `sentence_transformers.SentenceTransformer` — embedding model (optional)
- `numpy` — array assembly (optional)
- `sklearn.cluster.KMeans` — clustering (optional)

### Outbound (imports this file)
- **`reasons/derive.py`** — the primary consumer; calls `cluster_beliefs` or `cluster_beliefs_intra` before each LLM derivation round.
- **`reasons/contradictions.py`** — likely uses `list_clusters` to group beliefs for contradiction detection.
- **`reasons/api.py`** / **`reasons/cli.py`** — expose cluster-related flags (`--cluster`, `--exhaust`, `--n-clusters`) and wire them into `derive`.
- **`reasons_lib/cluster.py`** — appears to be a thin re-export shim (the lib mirror of the `reasons/` package).
- **Tests**: `test_cluster.py`, `test_derive.py`, `test_contradictions.py`, `test_api.py`, `test_cli.py`.

## 5. Flow (control path for `cluster_beliefs_intra`)

```
caller passes (beliefs, budget, round_num, seed, ...)
│
├─ _require_cluster_deps()          # raises ImportError if deps missing
│
├─ len(beliefs) <= budget?          # YES → return all, skip ML entirely
│
├─ cache = ClusterCache(...)        # load SentenceTransformer (expensive, once)
│
├─ ids, embeddings = cache.embed()  # encode only uncached texts, batch
│     │
│     ├─ sort keys for determinism
│     ├─ check (nid, sha256) cache
│     ├─ model.encode(uncached)     # one GPU/CPU forward pass
│     └─ assemble np.array
│
├─ k = _auto_k(n, n_clusters, ...)  # typically n//5, clamped [2, 20]
│
├─ KMeans(k, seed, n_init=10).fit_predict(embeddings)
│     → labels: np.ndarray of shape (n,)
│
├─ Build clusters dict: {label: [nid, ...]}
│
├─ sorted_labels = sorted by -len   # largest cluster first
├─ focus_label = sorted_labels[round_num % k]
│
├─ rng = random.Random(seed)
├─ selected = rng.sample(members_of_focus, min(budget, len(members)))
│
└─ return (selected, {n_clusters, cluster_sizes, embedding_model, focus_cluster})
```

The cross-cluster variant (`cluster_beliefs`) replaces the last sampling block with a **proportional allocation**: `base_per = budget // k`, remainder distributed to the largest clusters first, and `rng.sample` is applied per-cluster.

## 6. Invariants

- **Output size ≤ `budget`.** Every code path (early return, cross-cluster, intra-cluster) is capped. In cross-cluster, `alloc = min(alloc, len(members))` per cluster prevents over-sampling a small cluster. In intra-cluster, `alloc = min(budget, len(members))`.
- **`k` is always ≥ 1 and ≤ `len(beliefs)`.** `_auto_k` clamps: `max(k, 2)` then `min(k, n_beliefs)`. If `n_beliefs < 2`, the early-return shortcut fires first.
- **Deterministic given `(beliefs, budget, seed, n_clusters, model_name)`.** KMeans `random_state`, `random.Random(seed)`, and sorted iteration over keys guarantee this.
- **Embedding order matches `ids` order.** `cache.embed` returns `ids` (sorted) and the `np.array` in the same row order. KMeans labels align row-by-row with `ids`.
- **Cache is keyed by content, not just ID.** Changing the text of a belief invalidates its cache entry without invalidating others.
- **`list_clusters` never drops a belief.** It returns every belief in some cluster. The early-return threshold is `<= 3` (not `<= budget`) because this function has no budget — it's a full listing.

## 7. Error Handling

- **Missing optional deps** → `ImportError` with a specific pip command (`pip install 'ftl-reasons[cluster]'`). Raised by `_require_cluster_deps()` at the top of every public function. The stub assignments (`SentenceTransformer = None`, etc.) ensure that if someone accidentally accesses them, they get a clear `TypeError` rather than a confusing `NameError`.
- **No internal try/except.** If `model.encode()` or `KMeans.fit_predict()` raises (e.g., CUDA OOM, NaN embeddings), the exception propagates to the caller. There is no retry, fallback to a non-clustered selection, or logging. The caller (presumably `derive.py`) is expected to handle it.
- **No validation of `beliefs` contents.** If a value in the dict is not a string, `text.encode()` will raise `AttributeError`. If `beliefs` is empty, `len(beliefs) <= budget` is `True` (assuming `budget >= 0`) and the function returns `[]` immediately.
- **No thread-safety.** `ClusterCache._cache` is a plain dict. If two threads call `embed()` concurrently on the same cache, they may double-encode or see inconsistent intermediate state. The derive pipeline appears to be single-threaded.

## Topics to Explore

- [file] `reasons/derive.py` — The primary consumer of both `cluster_beliefs` and `cluster_beliefs_intra`; understanding how `round_num` advances and how `--exhaust` reuses `ClusterCache` across rounds reveals the full sampling strategy.
- [function] `reasons/cluster.py:cluster_beliefs` — Compare the cross-cluster proportional allocation loop against `cluster_beliefs_intra`'s single-cluster focus to understand the tradeoff between diversity and topical cohesion in derivation.
- [file] `tests/test_cluster.py` — Covers the edge cases: empty beliefs, budget ≥ n_beliefs, explicit `n_clusters`, cache hit/miss paths, and determinism with a fixed seed. Reading the tests clarifies intended invariants.
- [general] `sentence-transformers-model-selection` — `all-MiniLM-L6-v2` is a 384-dim, ~23M-param model. Understanding its embedding quality and how `model_name` can be overridden matters for tuning cluster quality vs. latency.
- [file] `reasons/contradictions.py` — Likely calls `list_clusters` to group topically related beliefs before checking pairwise contradictions, which changes the performance profile from O(n²) to per-cluster O(n²).

## Beliefs

- `cluster-intra-focus-rotates-by-round` — `cluster_beliefs_intra` selects the focus cluster as `sorted_by_size[round_num % k]`, so consecutive derive rounds cycle through clusters in deterministic order.
- `cluster-cache-keyed-by-text-hash` — The embedding cache uses `(node_id, sha256(text)[:16])` as its key, meaning a belief whose text is edited automatically gets a fresh embedding without invalidating other entries.
- `cluster-early-return-skips-ml` — All three public functions return immediately when `len(beliefs)` is at or below the threshold (≤ `budget` for sampling, ≤ 3 for listing), avoiding loading the SentenceTransformer model entirely.
- `cluster-requires-optional-extra` — `sentence-transformers` and `scikit-learn` are not hard dependencies; the module imports successfully without them, but any public function call raises `ImportError` directing the user to `pip install 'ftl-reasons[cluster]'`.
- `cross-cluster-allocates-proportionally` — `cluster_beliefs` distributes the budget as `budget // k` per cluster with the remainder given to the largest clusters first (sorted descending by size), then samples within each cluster.
