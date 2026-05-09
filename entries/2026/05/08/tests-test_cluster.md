# File: tests/test_cluster.py

**Date:** 2026-05-08
**Time:** 14:18

I'll work from the test file content you shared — the tests themselves reveal the full contract of the cluster API.

---

## Purpose

`test_cluster.py` is the test suite for `reasons_lib.cluster`, the semantic clustering module used in the derive pipeline. The derive pipeline generates new beliefs from existing ones; when the belief set is large, clustering groups semantically similar beliefs together so the pipeline can select a diverse, budget-constrained subset to reason over — rather than picking randomly or linearly.

This file validates:
1. The graceful-degradation path when optional ML dependencies aren't installed.
2. The core `cluster_beliefs` function: budget enforcement, reproducibility, cross-domain diversity, and stats output.
3. The `ClusterCache` embedding cache: hit behavior and incremental growth.

## Key Components

### Constants / Markers

- **`HAS_CLUSTER_DEPS`** (bool): Imported from `reasons_lib.cluster`. `True` when `sentence-transformers` and `scikit-learn` are available. If the import itself fails, the test module sets it to `False` locally.
- **`skip_no_cluster`**: A `pytest.mark.skipif` decorator applied to every test that needs the ML stack. This is the gate — tests skip cleanly on CI environments or minimal installs that don't have the `[cluster]` extra.

### Functions Under Test

| Function | Signature (inferred) | Contract |
|---|---|---|
| `cluster_beliefs` | `(beliefs: dict[str, str], budget: int, seed: int, n_clusters: int = None) -> (list[str], dict)` | Takes a `{belief_id: text}` dict, clusters by embedding similarity, selects up to `budget` items distributed across clusters, returns selected IDs and a stats dict. |
| `ClusterCache` | class | Caches embeddings keyed by belief text. `.embed(beliefs)` returns `(ids, embeddings)`. Re-embedding the same texts reuses cached vectors; new texts grow the cache incrementally. |
| `_require_cluster_deps` | `() -> None` | Guard function. Raises `ImportError` with a message mentioning `ftl-reasons[cluster]` when deps are missing. No-op when deps are present. |

### Stats Dict Shape

The stats dict returned by `cluster_beliefs` has at least:
- `n_clusters` (int) — number of clusters formed
- `cluster_sizes` (list[int]) — per-cluster member counts; `sum(cluster_sizes) == len(beliefs)`
- `embedding_model` (str) — name of the sentence-transformer model used

## Patterns

**Optional-dependency guard pattern.** The top-level `try/except ImportError` wrapping the import, combined with a module-level `skipif` marker, is a standard pattern for testing optional extras. It lets the test file load without crashing, and each test self-documents why it's skipped.

**Deterministic seeding.** Every `cluster_beliefs` call passes `seed=42`. This makes embedding → clustering → selection fully reproducible, which `test_cluster_beliefs_reproducible` explicitly verifies by calling twice and asserting list equality.

**Budget-as-contract.** Tests exercise both sides of the budget boundary: when there are fewer beliefs than the budget (`test_cluster_beliefs_under_budget` — 5 beliefs, budget 10 → all selected, 1 cluster), and when there are more (`test_cluster_beliefs_selects_budget` — 50 beliefs, budget 20 → exactly 20 selected, ≥2 clusters).

**Cross-domain diversity assertion.** `test_cluster_beliefs_cross_domain` creates two semantically distinct groups (auth vs. database) and asserts both are represented in the output. This validates that clustering + proportional selection actually achieves domain diversity, not just random sampling.

## Dependencies

**Imports:**
- `pytest` — test framework
- `reasons_lib.cluster` — the module under test (`cluster_beliefs`, `ClusterCache`, `_require_cluster_deps`, `HAS_CLUSTER_DEPS`)

**Transitive (via `reasons_lib.cluster`):**
- `sentence-transformers` — for embedding belief texts
- `scikit-learn` — for KMeans/similar clustering
- Both are optional (`[cluster]` extra in `pyproject.toml`)

**Imported by:** Nothing — it's a leaf test module.

## Flow

1. **Module load**: Try importing from `reasons_lib.cluster`. If that fails, set `HAS_CLUSTER_DEPS = False` so the skip marker works.
2. **Test collection**: pytest collects 9 test functions. 8 of them carry `@skip_no_cluster`, so they only run when the ML stack is present. One (`test_require_cluster_deps_message`) runs only when deps are *missing* — it self-skips otherwise.
3. **Test execution (happy path)**:
   - Build a synthetic `beliefs` dict (`{id: text}`)
   - Call `cluster_beliefs(beliefs, budget=N, seed=42)` or create a `ClusterCache` and call `.embed()`
   - Assert on the returned selected IDs, stats, or cache internals

## Invariants

- `cluster_beliefs` must return **exactly `budget`** items when `len(beliefs) > budget`, and **all items** when `len(beliefs) <= budget`.
- `cluster_beliefs` with the same inputs and same `seed` must return **identical** results (determinism).
- `sum(stats["cluster_sizes"])` must equal `len(beliefs)` — every belief belongs to exactly one cluster.
- When `n_clusters` is passed explicitly, the output `stats["n_clusters"]` must match it exactly.
- `ClusterCache.embed()` must not grow `_cache` when re-embedding previously seen texts.
- `ClusterCache.embed()` must grow `_cache` by exactly the number of new texts when given a superset.

## Error Handling

- **Missing deps**: `_require_cluster_deps()` raises `ImportError` with a message matching `ftl-reasons[cluster]`. The test verifies this regex.
- **Import failure at module level**: Caught silently — `HAS_CLUSTER_DEPS` falls back to `False`, and all dependent tests skip rather than error.
- No other error paths are tested — the module assumes valid inputs (dict of strings, positive integer budget).

## Topics to Explore

- [file] `reasons_lib/cluster.py` — The implementation of `cluster_beliefs` and `ClusterCache`; understanding the actual clustering algorithm (likely KMeans), how budget allocation is distributed across clusters, and which embedding model is used
- [file] `reasons_lib/derive.py` — The derive pipeline that consumes `cluster_beliefs` to select a diverse belief subset before LLM-driven derivation
- [function] `reasons_lib/cluster.py:cluster_beliefs` — How proportional selection works: does it round-robin across clusters, or allocate proportionally to cluster size?
- [file] `pyproject.toml` — The `[cluster]` optional extra definition, which controls whether `sentence-transformers` and `scikit-learn` are available
- [general] `derive-budget-mechanics` — How the budget parameter flows from CLI/config through derive into cluster selection, and what happens when clustering is unavailable (fallback behavior)

## Beliefs

- `cluster-beliefs-returns-exact-budget` — `cluster_beliefs` returns exactly `budget` belief IDs when the input set is larger than the budget, never more or fewer
- `cluster-beliefs-deterministic-with-seed` — Given the same beliefs dict, budget, and seed, `cluster_beliefs` produces identical output across calls
- `cluster-cache-no-recompute` — `ClusterCache.embed()` does not recompute embeddings for previously cached belief texts; cache size stays constant on repeated calls with the same input
- `cluster-stats-sizes-sum-to-input` — The `cluster_sizes` list in the stats dict always sums to the total number of input beliefs, enforcing that every belief is assigned to exactly one cluster
- `cluster-deps-optional-with-graceful-skip` — The clustering module is behind an optional `[cluster]` install extra; when deps are missing, `_require_cluster_deps` raises `ImportError` and all cluster tests skip cleanly

