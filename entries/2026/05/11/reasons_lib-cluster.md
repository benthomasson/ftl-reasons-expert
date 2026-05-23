# File: reasons_lib/cluster.py

**Date:** 2026-05-11
**Time:** 12:54

## `reasons_lib/cluster.py` — Semantic Clustering for Belief Selection

### Purpose

This module provides semantic clustering over the belief network to support **cross-domain sampling** in the derive pipeline. When `reasons derive` needs to select a subset of beliefs to feed an LLM for deriving new conclusions, naive random sampling tends to pull from the same topical neighborhood. This module embeds belief texts into vector space and clusters them with KMeans, then samples proportionally across clusters — ensuring the LLM sees beliefs from different conceptual regions and can form cross-domain connections.

It's an optional dependency (`pip install 'ftl-reasons[cluster]'`), gated behind a try/except import block.

### Key Components

**`ClusterCache`** — Caches sentence-transformer embeddings keyed by `(node_id, sha256_prefix)`. The sha256 component means if a belief's text changes, the stale embedding is bypassed and re-computed. Used in `--exhaust` mode where `cluster_beliefs` is called repeatedly across derive rounds, avoiding redundant GPU/CPU work on already-embedded beliefs.

**`cluster_beliefs(beliefs, budget, ...)`** — The main entry point for the derive pipeline. Takes a dict of `{node_id: text}` and a `budget` (max beliefs to return). It:
1. Embeds all beliefs
2. KMeans-clusters them into `k` groups
3. Allocates `budget` slots proportionally across clusters (largest clusters listed first for remainder distribution)
4. Randomly samples within each cluster up to the allocation

Returns `(selected_ids, cluster_stats)`.

**`list_clusters(beliefs, ...)`** — Diagnostic/introspection function. Clusters beliefs and returns the full cluster assignments without sampling. Used by the CLI's `reasons cluster` command to let users inspect how their belief space is organized.

**`_auto_k(n_beliefs, n_clusters, max_k)`** — Heuristic for choosing cluster count: `n_beliefs // 5`, clamped to `[2, max_k]`. The caller can override with an explicit `n_clusters`. In `cluster_beliefs`, `max_k` is further constrained to `budget // 3` so you don't end up with more clusters than you can meaningfully sample from.

**`_require_cluster_deps()`** — Guard that raises `ImportError` with installation instructions if sentence-transformers or scikit-learn aren't installed.

### Patterns

**Graceful optional dependency** — The try/except at module level sets `HAS_CLUSTER_DEPS = False` and stubs the imports to `None`. This lets the rest of the codebase import `cluster.py` unconditionally (for type checking, feature detection, etc.) without crashing. Actual usage is gated by `_require_cluster_deps()` at function entry.

**Content-addressed caching** — The cache key is `(node_id, sha256(text)[:16])`, not just `node_id`. This handles the case where a belief's text is edited between derive rounds — the old embedding won't match and will be recomputed.

**Proportional allocation with remainder distribution** — Budget is divided as `base_per = budget // k` with `remainder = budget % k` extra slots given to the largest clusters first (sorted by `-len`). This ensures the budget is fully used and larger clusters get slightly more representation.

### Dependencies

**Imports:**
- `sentence_transformers.SentenceTransformer` — text-to-vector embedding (the `all-MiniLM-L6-v2` model by default)
- `sklearn.cluster.KMeans` — clustering algorithm
- `numpy` — array operations for embeddings
- `random` — sampling within clusters
- `hashlib.sha256` — content-addressed cache keys

**Imported by (project-level):**
- `reasons_lib/derive.py` — uses `cluster_beliefs` and `ClusterCache` for cross-domain sampling during belief derivation
- `reasons_lib/contradictions.py` — uses clustering to select diverse beliefs for contradiction detection
- `reasons_lib/api.py` and `reasons_lib/cli.py` — expose clustering through the API and CLI respectively

### Flow

For `cluster_beliefs`:

```
beliefs dict → ClusterCache.embed()
  → split into cached vs uncached
  → batch-encode uncached with SentenceTransformer
  → merge back into ordered (ids, embeddings) array
→ _auto_k() → compute k
→ KMeans.fit_predict() → labels array
→ group ids by label → clusters dict
→ proportional budget allocation across clusters
→ rng.sample() within each cluster
→ (selected_ids, stats)
```

### Invariants

- **Budget is a hard ceiling**: `len(selected) <= budget` always. If `len(beliefs) <= budget`, all beliefs are returned without clustering.
- **k <= n_beliefs**: `_auto_k` enforces `min(k, n_beliefs)` at the end, so KMeans never gets more clusters than data points.
- **k >= 2** (when auto-computed): the heuristic floors at 2, ensuring at least two clusters for cross-domain sampling to be meaningful.
- **Deterministic given seed**: both KMeans (`random_state=seed`) and sampling (`random.Random(seed)`) use the same seed for reproducibility.
- **Sorted iteration**: `ids = sorted(beliefs.keys())` in `embed()` ensures consistent ordering between the ids list and embeddings array regardless of dict insertion order.

### Error Handling

The only explicit error path is `_require_cluster_deps()` raising `ImportError`. Beyond that, errors from KMeans or SentenceTransformer propagate uncaught — the module trusts its callers to pass valid data. The small-input guards (`len(beliefs) <= budget` in `cluster_beliefs`, `len(beliefs) <= 3` in `list_clusters`) serve as short-circuit fast paths rather than error handling.

---

## Topics to Explore

- [function] `reasons_lib/derive.py:derive` — Where `cluster_beliefs` is called; shows how budget, seed, and cache are threaded through multiple derive rounds in `--exhaust` mode
- [file] `tests/test_cluster.py` — Unit tests that document edge cases: empty input, budget exceeding belief count, seeded reproducibility
- [function] `reasons_lib/contradictions.py:find_contradictions` — Another consumer of clustering, using it to select diverse beliefs for contradiction scanning
- [file] `reasons_lib/api.py` — How `list_clusters` is exposed as an API endpoint and what parameters the CLI passes through
- [general] `cross-cluster-sampling-effectiveness` — Whether the proportional allocation strategy actually improves derive quality vs. pure random sampling

---

## Beliefs

- `cluster-cache-key-includes-text-hash` — ClusterCache keys embeddings by `(node_id, sha256(text)[:16])`, so modified belief text invalidates the cached embedding without requiring explicit cache eviction
- `cluster-auto-k-floors-at-two` — `_auto_k` always returns at least 2 clusters (when auto-computed), ensuring cross-domain sampling is meaningful even for small belief sets
- `cluster-deps-are-optional` — The module is importable without sentence-transformers or scikit-learn installed; the `ImportError` is deferred to function call time via `_require_cluster_deps()`
- `cluster-budget-is-hard-ceiling` — `cluster_beliefs` never returns more than `budget` belief IDs; when the belief set is smaller than budget, it returns all beliefs without clustering
- `cluster-remainder-favors-largest` — When budget doesn't divide evenly across clusters, extra slots are allocated to the largest clusters first (sorted by descending size)

