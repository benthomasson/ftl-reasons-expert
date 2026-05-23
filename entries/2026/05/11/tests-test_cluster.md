# File: tests/test_cluster.py

**Date:** 2026-05-11
**Time:** 12:55

I'll work from the test file content and the imports visible there.

---

## Purpose

`tests/test_cluster.py` validates the **semantic clustering subsystem** in `reasons_lib.cluster`. This module groups beliefs by semantic similarity using sentence embeddings and k-means clustering, enabling the derive pipeline to select a diverse, budget-constrained subset of beliefs for LLM-driven derivation. The test file ensures that clustering, budget selection, caching, reproducibility, and the `list_clusters` inspection API all behave correctly.

## Key Components

### Skip infrastructure

- **`HAS_CLUSTER_DEPS`** — boolean imported from `reasons_lib.cluster` (or set to `False` if the import itself fails). Guards whether clustering dependencies (`sentence-transformers`, `scikit-learn`) are available.
- **`skip_no_cluster`** — a `pytest.mark.skipif` decorator applied to every test that needs the ML dependencies. Tests degrade gracefully on minimal installs.

### Test functions

| Test | What it verifies |
|------|-----------------|
| `test_require_cluster_deps_message` | When deps are missing, `_require_cluster_deps()` raises `ImportError` with the install command `ftl-reasons[cluster]`. Only runs when deps are *not* installed. |
| `test_cluster_beliefs_under_budget` | When `len(beliefs) <= budget`, all beliefs are returned and clustering collapses to a single cluster. |
| `test_cluster_beliefs_selects_budget` | With 50 beliefs and `budget=20`, exactly 20 are selected across ≥2 clusters. |
| `test_cluster_beliefs_reproducible` | Same `seed` produces identical selections across two calls. |
| `test_cluster_beliefs_cross_domain` | With two semantically distinct groups ("auth" vs "db"), the budget selection draws from both — verifying diversity, not just random sampling. |
| `test_cluster_cache_reuse` | `ClusterCache.embed()` called twice with the same beliefs doesn't grow the cache — embeddings are reused. |
| `test_cluster_cache_incremental` | Adding 5 new beliefs to an existing set grows the cache by exactly 5 — only new texts are embedded. |
| `test_cluster_stats_shape` | The `stats` dict returned by `cluster_beliefs` contains `n_clusters`, `cluster_sizes`, `embedding_model`, and cluster sizes sum to the total belief count. |
| `test_cluster_n_clusters_override` | Passing `n_clusters=5` forces exactly 5 clusters regardless of heuristic. |
| `test_list_clusters_returns_all_beliefs` | `list_clusters` partitions *all* beliefs into clusters (no drops, no duplicates). |
| `test_list_clusters_n_clusters_override` | `n_clusters` override works for `list_clusters` too. |
| `test_list_clusters_small_set` | With only 2 beliefs, everything collapses to 1 cluster. |
| `test_list_clusters_reproducible_with_seed` | Deterministic ordering with fixed seed. |
| `test_auto_k_defaults` | `_auto_k` applies the `n // 5` heuristic with a floor of 2. |
| `test_auto_k_with_override` | Explicit `n_clusters` overrides heuristic, clamped to `n`. |
| `test_auto_k_with_max_k` | `max_k` caps the computed cluster count. |

## Patterns

1. **Optional-dependency gating** — The entire import is wrapped in `try/except ImportError`. A module-level `skip_no_cluster` marker is applied to individual tests rather than the whole module, so that the `_auto_k` tests and the missing-dep error-message test can still run without ML packages.

2. **Dual-path testing** — `test_require_cluster_deps_message` uses `pytest.skip` when deps *are* installed, testing the failure path only when it's actually reachable. This avoids mocking.

3. **Seed-based determinism** — Tests that verify reproducibility pass `seed=42` to `cluster_beliefs`/`list_clusters`. This contracts the implementation to accept and honor a `seed` parameter for the underlying k-means.

4. **Stats-as-contract** — Several tests assert the shape and contents of the `stats` dict, treating it as a stable API surface (keys `n_clusters`, `cluster_sizes`, `embedding_model`).

5. **Cross-domain diversity assertion** — `test_cluster_beliefs_cross_domain` constructs beliefs with intentionally distinct semantic content ("auth"/"login" vs "database"/"indexing") and asserts both domains appear in the selection. This is a semantic integration test: it exercises the embedding model's ability to distinguish topics.

## Dependencies

**Imports:**
- `pytest` — test framework
- `reasons_lib.cluster` — the module under test: `cluster_beliefs`, `list_clusters`, `ClusterCache`, `_require_cluster_deps`, `_auto_k`, `HAS_CLUSTER_DEPS`

**Transitive (inside `reasons_lib.cluster`):**
- `sentence-transformers` — for embedding belief text
- `scikit-learn` — for k-means clustering
- Both are optional extras (`ftl-reasons[cluster]`)

**Imported by:** Nothing — this is a leaf test module.

## Flow

1. At import time, the module attempts to import from `reasons_lib.cluster`. If that fails, `HAS_CLUSTER_DEPS = False` and all `@skip_no_cluster`-decorated tests are skipped.

2. For the `cluster_beliefs` tests, the flow is:
   - Construct a `dict[str, str]` mapping belief IDs to text
   - Call `cluster_beliefs(beliefs, budget=N, seed=42)`
   - Assert properties of the returned `(selected_ids, stats)` tuple

3. For `ClusterCache` tests, the flow exercises the embed-and-cache cycle:
   - Create a `ClusterCache` instance
   - Call `cache.embed(beliefs)` → returns `(ids, embeddings)`
   - Verify `cache._cache` size to confirm caching/incremental behavior

4. `_auto_k` tests are pure function tests — no ML, no I/O.

## Invariants

- **Budget is exact:** When `len(beliefs) > budget`, `cluster_beliefs` returns exactly `budget` beliefs (asserted in `test_cluster_beliefs_selects_budget`).
- **Budget ≥ n → all returned:** When beliefs fit within budget, all are returned (asserted in `test_cluster_beliefs_under_budget`).
- **No beliefs lost in clustering:** `list_clusters` partitions all input beliefs across clusters — the union of all cluster members equals the input set (asserted in `test_list_clusters_returns_all_beliefs`).
- **Cluster sizes sum to n:** `sum(stats["cluster_sizes"]) == len(beliefs)` (asserted in `test_cluster_stats_shape`).
- **`_auto_k` floor is 2:** Even for small inputs, at least 2 clusters are requested (asserted with `_auto_k(5) == 2` and `_auto_k(10) == 2`).
- **`n_clusters` override clamped to n:** `_auto_k(3, n_clusters=10) == 3` — you can't have more clusters than beliefs.

## Error Handling

- **Missing dependencies:** `_require_cluster_deps()` raises `ImportError` with a message containing `ftl-reasons[cluster]`, tested by `test_require_cluster_deps_message`.
- **No other error paths tested:** The tests assume valid inputs (non-empty dicts, positive budgets). There are no tests for empty-input edge cases or malformed belief dicts.

---

## Topics to Explore

- [file] `reasons_lib/cluster.py` — The implementation: embedding model selection, k-means strategy, how budget selection picks representatives from each cluster
- [function] `reasons_lib/derive.py:derive` — How the derive pipeline calls `cluster_beliefs` to select a diverse subset before sending to the LLM
- [general] `_auto_k-heuristic` — The `n // 5` with floor-2 and max-k capping strategy — understand whether this heuristic scales for large belief sets (500+)
- [file] `tests/test_derive_budget.py` — Tests for budget enforcement in the derive pipeline, likely exercises clustering as part of the end-to-end flow
- [general] `ClusterCache-lifecycle` — Where `ClusterCache` instances are created and reused across derive rounds — determines whether embedding work is amortized across calls

## Beliefs

- `cluster-beliefs-returns-exact-budget` — When the number of beliefs exceeds the budget, `cluster_beliefs` returns exactly `budget` belief IDs, not fewer or more.
- `cluster-deps-are-optional-extras` — `sentence-transformers` and `scikit-learn` are optional; the cluster module degrades gracefully via `HAS_CLUSTER_DEPS` and `_require_cluster_deps()`.
- `auto-k-floor-is-two` — `_auto_k` never returns fewer than 2 clusters, regardless of input size, unless clamped by `n` itself.
- `cluster-cache-is-incremental` — `ClusterCache.embed()` only computes embeddings for texts not already in the cache, growing by exactly the number of new entries.
- `list-clusters-partitions-all-beliefs` — `list_clusters` assigns every input belief to exactly one cluster with no drops or duplicates.

