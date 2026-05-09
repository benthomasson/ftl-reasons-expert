# File: reasons_lib/cluster.py

**Date:** 2026-05-08
**Time:** 14:15



## `reasons_lib/cluster.py` — Semantic Clustering for Belief Selection

### Purpose

This module solves a specific problem in the `derive` pipeline: when you have more IN premises than your LLM context budget allows, how do you pick a representative subset? Rather than random sampling (which might over-represent one topic area), it embeds belief texts into vector space using sentence-transformers, clusters them with KMeans, and then samples evenly across clusters. This ensures the derive step sees beliefs from across the full semantic range of the knowledge base.

It's an optional feature — the heavy ML dependencies (sentence-transformers, scikit-learn, numpy) are behind a lazy import guard and an extras group (`ftl-reasons[cluster]`).

### Key Components

**`HAS_CLUSTER_DEPS`** (module-level bool): Gate that tracks whether the optional dependencies loaded. Everything in this module checks this before doing real work.

**`_require_cluster_deps()`**: Raises a clear `ImportError` with install instructions if the optional deps are missing. Called at the top of both `ClusterCache.__init__` and `cluster_beliefs`, so you fail fast with a helpful message rather than hitting a cryptic `NoneType` error.

**`ClusterCache`**: Caches embedding vectors across multiple derive rounds, keyed by `(node_id, sha256_prefix_of_text)`. This matters for `--exhaust` mode where derive runs repeatedly — re-embedding hundreds of beliefs each round would be wasteful. The cache key includes a content hash so that if a belief's text changes (same ID, updated wording), it gets re-embedded rather than serving stale vectors.

- `embed(beliefs)` takes a `{node_id: text}` dict, computes embeddings only for uncached entries, and returns `(ids_list, numpy_array)` in sorted-by-id order. The sorted ordering is important — it makes clustering deterministic given the same seed.

**`cluster_beliefs(beliefs, budget, ...)`**: The main entry point. Takes a dict of belief texts and a budget cap, returns `(selected_ids, cluster_stats)`.

### Patterns

**Optional dependency guard**: The try/except at module level with a `HAS_CLUSTER_DEPS` sentinel is a standard Python pattern for optional features. The fallback assigns `None` to the names so type checkers and IDE navigation still work in the no-deps case.

**Cache invalidation by content hash**: Using `sha256(text)[:16]` as part of the cache key is a pragmatic content-addressing scheme. The 16-hex-char prefix gives 64 bits of collision resistance — plenty for a cache that lives within a single process.

**Round-robin allocation with remainder distribution**: The sampling logic divides the budget evenly across `k` clusters (`base_per = budget // k`), then distributes the remainder one-per-cluster to the largest clusters first (via `sorted_labels` sorted by descending size). This avoids both starvation of small clusters and waste on clusters too small to fill their allocation.

### Dependencies

**Imports**: `random` and `hashlib.sha256` from stdlib. Optionally `sentence_transformers.SentenceTransformer`, `numpy`, and `sklearn.cluster.KMeans`.

**Imported by** (project-relevant only):
- `reasons_lib/derive.py` — uses `cluster_beliefs` and `ClusterCache` to select premises when `--cluster` is passed
- `reasons_lib/cli.py` — wires the `--cluster` flag through to derive
- `tests/test_cluster.py`, `tests/test_derive.py` — test coverage

### Flow

1. **Early exit**: If `len(beliefs) <= budget`, skip all ML work and return everything.
2. **Embed**: Build or reuse a `ClusterCache`, call `cache.embed(beliefs)` to get the embedding matrix.
3. **Choose k**: If no explicit `n_clusters`, auto-compute as `len(beliefs) // 5`, clamped between 2 and `min(budget // 3, 20)`. This heuristic targets ~5 beliefs per cluster and ensures at least 3 beliefs per cluster given the budget.
4. **Cluster**: Run KMeans with `n_init=10` (sklearn's robust default) and the user's seed for reproducibility.
5. **Sample**: Allocate budget across clusters proportionally, sample within each cluster randomly.
6. **Return**: The selected IDs plus stats dict (cluster count, sizes, model name) for logging/debugging.

### Invariants

- `len(selected) <= budget` always holds — each cluster's allocation is capped by `min(alloc, len(members))`.
- Embedding order is deterministic: `ids = sorted(beliefs.keys())` before any embedding or clustering.
- The cache key includes content, not just ID — editing a belief's text forces re-embedding.
- `k` is clamped to `[1, len(beliefs)]` so KMeans never receives an invalid cluster count.

### Error Handling

Minimal and intentional. The only explicit error is the `ImportError` from `_require_cluster_deps()`. Beyond that, the code relies on the ML libraries' own validation (e.g., KMeans will raise if given degenerate input). There's no try/except around the embedding or clustering calls — failures there represent genuine bugs or environment problems that should propagate.

---

## Topics to Explore

- [file] `reasons_lib/derive.py` — The consumer of `cluster_beliefs`; shows how cluster mode integrates with the LLM derivation loop and `--exhaust` mode
- [file] `tests/test_cluster.py` — Covers edge cases like budget >= belief count, deterministic seeding, and cache invalidation
- [function] `reasons_lib/derive.py:derive` — Where `ClusterCache` is instantiated and reused across rounds
- [general] `kmeans-k-selection-heuristic` — The `len // 5, clamped to budget // 3` formula is empirical; worth understanding whether it holds for belief sets much larger than ~100
- [file] `reasons_lib/cli.py` — How `--cluster` and `--n-clusters` flags are parsed and forwarded

## Beliefs

- `cluster-beliefs-respects-budget` — `cluster_beliefs` never returns more IDs than the `budget` parameter; each cluster allocation is individually capped by `min(alloc, len(members))`
- `cluster-cache-keys-include-content-hash` — `ClusterCache` keys embeddings by `(node_id, sha256_prefix)`, so editing a belief's text with the same ID forces re-embedding rather than serving stale vectors
- `cluster-deps-are-optional` — `sentence-transformers` and `scikit-learn` are optional dependencies; the module degrades to a clear `ImportError` with install instructions when absent
- `cluster-embed-order-is-deterministic` — Beliefs are sorted by ID before embedding, making cluster assignments reproducible given the same random seed
- `cluster-remainder-favors-largest` — When the budget doesn't divide evenly across clusters, extra slots go to the largest clusters first (descending size sort)

