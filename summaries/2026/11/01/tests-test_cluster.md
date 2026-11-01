---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_cluster.py

# `tests/test_cluster.py` — Semantic Clustering Tests

## Purpose

This file is the test suite for the **semantic clustering** subsystem in the `ftl-reasons` derive pipeline. Its job is to verify that when the system needs to reduce a large set of beliefs down to a manageable budget (e.g., picking 20 representative beliefs from 50), the selection is:

- **Correct** — respects the budget, covers diverse topics, and returns all beliefs when the budget is generous enough
- **Deterministic** — same inputs + seed always produce the same output
- **Efficient** — the `ClusterCache` avoids redundant embedding computations
- **Well-behaved at the edges** — tiny belief sets, explicit `n_clusters` overrides, and the `_auto_k` formula all behave as documented

The file owns the contract of `reasons.cluster`'s public API (`cluster_beliefs`, `list_clusters`, `ClusterCache`) and its internal helper (`_auto_k`).

## Key Components

| Symbol | Kind | Contract |
|---|---|---|
| `HAS_CLUSTER_DEPS` | module-level flag | `True` if `sentence-transformers` + `scikit-learn` are importable; `False` otherwise. Set inside the `try/except` at the top of the file. |
| `skip_no_cluster` | `pytest.mark.skipif` | Decorator that skips a test when `HAS_CLUSTER_DEPS` is `False`. Used on every test that actually exercises clustering. |
| `cluster_beliefs(beliefs, budget, seed, n_clusters=None)` | function under test | Takes a `dict[str, str]` of belief-id → text. Returns `(selected: dict[str, str], stats: dict)`. `selected` has at most `budget` entries. `stats` must contain `n_clusters`, `cluster_sizes` (list of ints summing to `len(beliefs)`), and `embedding_model`. |
| `list_clusters(beliefs, seed=None, n_clusters=None)` | function under test | Returns `{"clusters": [{"beliefs": [{"id": …}]}, …], "n_clusters": int, "embedding_model": str}`. Every input belief ID must appear in exactly one cluster. |
| `ClusterCache` | class under test | Provides `.embed(beliefs) → (ids, embeddings)`. Internally caches embeddings in `self._cache`. Repeated calls with the same beliefs must **not** grow the cache; adding new beliefs must grow it by exactly the number of new entries. |
| `_auto_k(n_beliefs, n_clusters=None, max_k=None)` | helper under test | Computes a default number of clusters. Observed formula from tests: `min(n_beliefs // 5, 20)` (100→20, 50→10, 10→2, 5→2). Overridden by explicit `n_clusters` (capped at `n_beliefs`), and capped by `max_k`. |
| `_require_cluster_deps` | function under test | Raises `ImportError` with a message containing the string `ftl-reasons[cluster]` when optional deps are missing. |

## Patterns

1. **Optional-dependency guard pattern.** The file wraps the import in `try/except ImportError` and sets `HAS_CLUSTER_DEPS = False` as a fallback. This is the standard Python idiom for "this feature is in an extra (`pip install 'ftl-reasons[cluster]'`) and may not be present." Every clustering test is decorated with `@skip_no_cluster`, so the suite is **green even in a minimal install** — you never get a hard `ImportError` at collection time.

2. **Seed-based determinism testing.** Two tests (`test_cluster_beliefs_reproducible`, `test_list_clusters_reproducible_with_seed`) call the API twice with `seed=42` and assert exact equality. This guards against non-determinism from KMeans initialization or numpy shuffling.

3. **Budget-as-contract testing.** The pair `test_cluster_beliefs_under_budget` and `test_cluster_beliefs_selects_budget` bracket the two regimes: `budget ≥ n` (identity) and `budget < n` (exact selection). Together they pin down the boundary behavior.

4. **Cross-domain coverage assertion.** `test_cluster_beliefs_cross_domain` constructs two clearly different topic families (auth vs. DB) and asserts that *both* appear in the 20 selected from 50. This is a behavioral guard against a degenerate clustering that lumps everything into one blob.

5. **Cache invariant testing.** `test_cluster_cache_reuse` and `test_cluster_cache_incremental` directly inspect the private `cache._cache` to verify:
   - No duplicate entries on repeat calls.
   - Growth is exactly `+5` when 5 new beliefs are added.

6. **`_auto_k` table-driven tests.** The last three tests (`test_auto_k_defaults`, `test_auto_k_with_override`, `test_auto_k_with_max_k`) are pure unit tests of the formula with no model loading. They run **regardless** of whether cluster deps are installed (note: no `@skip_no_cluster`), because `_auto_k` is a simple arithmetic function that doesn't need sentence-transformers.

## Dependencies

**This file imports:**
- `pytest` — test framework
- `reasons.cluster` — the module under test (`cluster_beliefs`, `list_clusters`, `ClusterCache`, `_require_cluster_deps`, `_auto_k`, `HAS_CLUSTER_DEPS`)

**Transitively (when installed):**
- `sentence-transformers` — embedding model
- `scikit-learn` — KMeans (or similar) clustering

**Nothing in the repo imports this test file.** It is a leaf in the dependency graph. The CI workflow (`.github/workflows/`) presumably runs `pytest tests/` which discovers it via filename convention.

## Flow

```
Module load
  │
  ├─ try: from reasons.cluster import (… , HAS_CLUSTER_DEPS)
  │  except ImportError: HAS_CLUSTER_DEPS = False
  │
  └─ define skip_no_cluster = pytest.mark.skipif(not HAS_CLUSTER_DEPS, …)

pytest collection & execution
  │
  ├─ test_require_cluster_deps_message
  │    if HAS_CLUSTER_DEPS → skip (can't simulate missing deps)
  │    else → call _require_cluster_deps(), expect ImportError with "ftl-reasons[cluster]"
  │
  ├─ @skip_no_cluster tests (all skipped if deps absent)
  │    │
  │    ├─ Create beliefs: {id: text} dict (5–50 items)
  │    ├─ Call cluster_beliefs(beliefs, budget=B, seed=42[, n_clusters=K])
  │    ├─ Assert on len(selected), stats keys, cross-domain presence, reproducibility
  │    │
  │    ├─ Call list_clusters(beliefs, …)
  │    ├─ Assert every belief ID is in exactly one cluster, n_clusters correct
  │    │
  │    ├─ Instantiate ClusterCache, call .embed() twice
  │    └─ Assert cache._cache size is stable / grows by exact delta
  │
  └─ _auto_k tests (always run, no deps needed)
       └─ Assert _auto_k(n) for n ∈ {5, 10, 50, 100}, plus override and max_k cases
```

There is no shared fixture or state between tests — each test builds its own belief dict inline. The only shared state is the module-level `HAS_CLUSTER_DEPS` flag.

## Invariants

1. **Budget upper bound.** `len(cluster_beliefs(beliefs, budget=B)[0]) ≤ B` always. When `B ≥ len(beliefs)`, the result is the identity set with exactly 1 cluster.
2. **Reproducibility.** `cluster_beliefs(X, budget, seed=S)` is a pure function of `(X, budget, S)` — two calls with the same triple must return identical dicts.
3. **Cross-domain representation.** With two disjoint topic groups and a budget that is a subset of the total, the selected set must include members of *both* groups (not all from one).
4. **Cache idempotency.** `cache.embed(B)` called twice with the same `B` must not increase `len(cache._cache)`.
5. **Cache additivity.** `cache.embed(B ∪ N)` after `cache.embed(B)` grows the cache by exactly `|N|`.
6. **Stats completeness.** `stats` always contains `n_clusters` (int), `cluster_sizes` (list of ints whose sum equals `len(beliefs)`), and `embedding_model` (str).
7. **`_auto_k` formula.** Without overrides, `_auto_k(n) = min(n // 5, 20)` with a floor of 2. With `n_clusters=K`, result is `min(K, n)`. With `max_k=M`, result is capped at `M`.
8. **`list_clusters` completeness.** Every input belief ID appears in exactly one cluster across the full result.

## Error Handling

- **Missing optional deps** is the only error path explicitly tested. `_require_cluster_deps` raises `ImportError` with a user-facing message that includes the pip install command (`ftl-reasons[cluster]`). The test `test_require_cluster_deps_message` asserts the message matches the regex `ftl-reasons\[cluster\]`.
- The `try/except ImportError` at module level **swallows** the import failure so that test collection doesn't crash in a minimal install; it merely sets `HAS_CLUSTER_DEPS = False`.
- No tests exercise runtime errors *within* clustering (e.g., model download failure, NaN embeddings). Those would be integration-level concerns.
- There is no `conftest.py`-level mock or fixture for the embedding model — tests that run assume the real `sentence-transformers` model is loadable.

## Topics to Explore

- [file] `reasons/cluster.py` — The module under test. Understanding `cluster_beliefs`, `ClusterCache.embed`, and the actual `_auto_k` implementation (and how it calls the embedding model) is essential to interpreting why these assertions pass or fail.
- [file] `reasons/derive.py` — The derive pipeline that calls into clustering. Knowing *where* and *why* `cluster_beliefs` is invoked (what triggers it, what budget is chosen) gives context for what the tests are really guarding.
- [file] `pyproject.toml` — Defines the `[cluster]` optional extra (`sentence-transformers`, `scikit-learn`). Worth checking the pinned versions and whether the extra is listed in CI's install step.
- [file] `tests/conftest.py` — May contain shared fixtures (e.g., a mock embedding model or a small in-memory store) that affect how these tests run in CI versus locally.
- [general] `auto-k-selection-strategy` — The `_auto_k` formula (`n // 5`, capped at 20) is a heuristic. Understanding the product rationale behind "one representative per ~5 beliefs" helps decide whether the tests are pinning the right behavior or should be parameterized differently.

## Beliefs

- `test-cluster-skips-without-extras` — The entire clustering test suite (except `_auto_k` tests and the missing-dep message test) is skipped when `sentence-transformers` or `scikit-learn` are not installed, so a minimal `pip install ftl-reasons` yields a fully green `pytest tests/test_cluster.py`.
- `cluster-beliefs-respects-budget-exactly` — When `budget < len(beliefs)`, `cluster_beliefs` returns exactly `budget` beliefs, never fewer, and `stats["n_clusters"] ≥ 2` for sufficiently diverse inputs.
- `auto-k-is-pure-arithmetic` — `_auto_k` performs no I/O, model loading, or embedding; it is a deterministic function of `(n_beliefs, n_clusters, max_k)` and is testable without the `[cluster]` extra installed.
- `cluster-cache-is-keyed-by-belief-id` — `ClusterCache._cache` grows by the count of *new* belief IDs, not by the total number of beliefs passed to `.embed()`, implying the cache key is the belief ID (or its text) and re-embedding an existing ID is a no-op.
- `stats-cluster-sizes-sum-to-input-size` — In both `cluster_beliefs` and `list_clusters`, the sum of `cluster_sizes` (or the total beliefs across all cluster entries) always equals `len(beliefs)`, enforcing that no belief is silently dropped or duplicated.
