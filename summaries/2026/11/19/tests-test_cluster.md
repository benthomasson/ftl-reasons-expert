---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_cluster.py

# tests/test_cluster.py — Explanation

## Purpose

This is a **contract/behavior test suite** for `reasons/cluster.py`, the module that performs semantic clustering of beliefs in the `derive` pipeline (see the module docstring). It owns three jobs:

1. Verify the clustering API (`cluster_beliefs`, `list_clusters`) behaves per contract — budget adherence, determinism, diversity, stats shape.
2. Verify `ClusterCache` is actually a cache — reuse and incremental growth.
3. Gate the whole thing on optional dependencies (`sentence-transformers` + `scikit-learn`) so the suite can be collected and run without the `ftl-reasons[cluster]` extra installed.

It is a **leaf test file** — nothing imports it; it only depends on `reasons.cluster` and `pytest`.

## Key Components

### Module-level flag & import guard (top of file)
```python
try:
    from reasons.cluster import (cluster_beliefs, list_clusters, ClusterCache,
                                 _require_cluster_deps, _auto_k, HAS_CLUSTER_DEPS)
except ImportError:
    HAS_CLUSTER_DEPS = False
```
- `HAS_CLUSTER_DEPS` is the single source of truth for whether the heavy tests can run. It is either *imported from `reasons.cluster`* (that module exposes it) or set to `False` if even the module import fails.
- The `except ImportError` branch leaves the six names above **unbound** in the test module. This is why `test_require_cluster_deps_message` re-imports `_require_cluster_deps` locally rather than relying on the top-level binding — and why the `_auto_k` tests (undecorated) would `NameError` on a truly broken install, surfacing the misconfiguration instead of silently skipping.

### `skip_no_cluster` marker
```python
skip_no_cluster = pytest.mark.skipif(not HAS_CLUSTER_DEPS,
                                     reason="sentence-transformers and scikit-learn not installed")
```
Decorates 12 of the 16 tests. The classic optional-dependency idiom.

### `test_require_cluster_deps_message` — inverse-skip test
Only runs when deps are **missing** (`pytest.skip` otherwise). Asserts `_require_cluster_deps()` raises `ImportError` matching `ftl-reasons\[cluster\]`. This is a *documentation-as-test*: it guarantees the error message tells users how to fix the problem.

### `cluster_beliefs(beliefs, budget, seed, n_clusters=None) -> (selected_ids, stats)`
Contracts exercised:
- **`test_cluster_beliefs_under_budget`**: `budget >= len(beliefs)` → all IDs returned and `stats["n_clusters"] == 1`. So there is a short-circuit path that bypasses clustering entirely when the budget covers the corpus.
- **`test_cluster_beliefs_selects_budget`**: `len(selected) == 20` for a 50-belief corpus with `budget=20` — selection is *exactly* the budget, not ≤.
- **`test_cluster_beliefs_reproducible`**: same `(beliefs, budget, seed)` → identical `selected` list (order included).
- **`test_cluster_beliefs_cross_domain`**: with 25 auth + 25 DB beliefs and `budget=20`, both domains appear in the selection — the sampler must be cluster-aware, not a global top-K.
- **`test_cluster_beliefs_n_clusters_override`**: `n_clusters=5` is reflected in `stats["n_clusters"]`.

### `list_clusters(beliefs, n_clusters=None, seed=?) -> dict`
Returns:
```python
{
  "clusters": [ { "beliefs": [ {"id": ..., ...}, ... ] }, ... ],
  "n_clusters": int,
  "embedding_model": str,
}
```
Contracts:
- **`test_list_clusters_returns_all_beliefs`**: the flat set of `b["id"]` equals the input key set — a *partition*, every belief appears in exactly one cluster, none lost.
- **`test_list_clusters_n_clusters_override`**: `n_clusters=5` → exactly 5 clusters in `clusters`.
- **`test_list_clusters_small_set`**: only 2 beliefs → returns 1 cluster, but note this is *not* what `_auto_k(2)` would yield (`_auto_k(5) == 2` per `test_auto_k_defaults`), so there is an additional small-corpus guard inside `list_clusters`.
- **`test_list_clusters_reproducible_with_seed`**: same `seed` → deterministic cluster and member ordering.

### `ClusterCache`
Public class with `embed(beliefs) -> (ids, embeddings)` and an internal `_cache` dict keyed presumably by belief id.
- **`test_cluster_cache_reuse`**: a second `embed` on the same dict leaves `len(cache._cache)` unchanged and returns identical `ids`.
- **`test_cluster_cache_incremental`**: embedding 10 then 15 (5 new) beliefs grows `_cache` by exactly 5 — new items are embedded, cached ones are reused.

### `_auto_k(n, n_clusters=None, max_k=None) -> int` — pure-logic tests (never skipped)
- `_auto_k(100)==20`, `_auto_k(50)==10`, `_auto_k(10)==2`, `_auto_k(5)==2` → default is `max(2, n // 5)`.
- `_auto_k(100, n_clusters=5)==5`, `_auto_k(3, n_clusters=10)==3` → explicit `n_clusters` wins but is clamped to `n` (never more clusters than points).
- `_auto_k(100, max_k=8)==8`, `_auto_k(100, max_k=30)==20` → `max_k` caps the *auto-derived* value, not the explicit override.

These three tests are deliberately undecorated — they test pure arithmetic and should pass on any machine.

### `test_cluster_stats_shape`
Locks in the stats contract:
- keys `n_clusters`, `cluster_sizes` (a `list`), `embedding_model`;
- `sum(stats["cluster_sizes"]) == len(beliefs)` — every embedding is accounted for.

## Patterns

- **Optional-extras gating**: a single `HAS_CLUSTER_DEPS` flag drives `pytest.mark.skipif`; the module is importable even when the heavy deps are missing.
- **Contract testing over implementation testing**: assertions are on behavior (exact budget, partition invariant, determinism, cross-domain coverage) rather than on internal shapes — with one deliberate exception (`ClusterCache._cache` length) to prove caching is real.
- **Inverse skip**: `test_require_cluster_deps_message` skips when deps *are* present, the mirror of every other test — this makes the missing-dep path itself testable.
- **Determinism via explicit seed**: every stochastic test accepts `seed=42` and asserts identical output on repeat.
- **Divide-and-conquer fixtures**: no `conftest.py` fixtures; each test builds a small synthetic corpus in-line. Cheap, isolated, no shared state.

## Dependencies

**Imports**
- `pytest` — markers, `raises`, `skip`.
- `reasons.cluster` — the entire surface under test: `cluster_beliefs`, `list_clusters`, `ClusterCache`, `_require_cluster_deps`, `_auto_k`, `HAS_CLUSTER_DEPS`.

**Transitive runtime deps** (only when the extra is installed): `sentence-transformers` (embeddings), `scikit-learn` (KMeans or similar). Declared as `ftl-reasons[cluster]` in `pyproject.toml` per the docstring.

**Imported by**: nothing — it is a leaf test module.

## Flow

1. **Collection**: pytest imports the module. The top-level `try/except` determines `HAS_CLUSTER_DEPS`; `skip_no_cluster` is defined.
2. **Per-test dispatch**:
   - `test_require_cluster_deps_message` runs only when `HAS_CLUSTER_DEPS is False`, re-imports `_require_cluster_deps` locally, and asserts the `ImportError` message regex.
   - All `@skip_no_cluster` tests skip when deps are missing, otherwise build a synthetic `{id: text}` dict, call the API, and assert.
   - The three `_auto_k` tests always run (pure arithmetic, no skip marker).
3. **Data pushed through the pipeline**:
   - `cluster_beliefs`: `{id: text}` → embeddings → cluster → per-cluster sample → `(list_of_ids, stats)`.
   - `list_clusters`: `{id: text}` → embeddings → cluster → `{clusters, n_clusters, embedding_model}`.
   - `ClusterCache.embed`: `{id: text}` → `(list_of_ids, similarity_matrix_or_vectors)`.

## Invariants

- **Budget exactness**: when `0 < budget < len(beliefs)`, `len(selected) == budget`. Not "≤ budget".
- **Under-budget short-circuit**: `budget >= len(beliefs)` ⇒ all IDs returned with `n_clusters == 1`.
- **Determinism**: identical inputs + identical `seed` ⇒ identical output *including ordering* for both `cluster_beliefs` and `list_clusters`.
- **Partition invariant**: `list_clusters` returns every input id exactly once across all clusters; `sum(stats["cluster_sizes"]) == len(beliefs)` for `cluster_beliefs`.
- **Cross-cluster diversity**: the selection over a mixed-domain corpus must touch both domains — the algorithm may not degenerate to a single-cluster top-K.
- **`n_clusters` override fidelity**: an explicit `n_clusters` is honored exactly by both functions (subject to the small-corpus guard).
- **`_auto_k` arithmetic**: `default = max(2, n // 5)`, `max_k` caps the default, `n_clusters` overrides and is clamped to `n`.
- **Cache monotonicity**: `ClusterCache` never re-embeds a belief already in `_cache`; adding `k` new ids grows `_cache` by exactly `k`.

## Error Handling

- **Missing-deps path**: `_require_cluster_deps()` raises `ImportError` with a message containing `ftl-reasons[cluster]` — this is the guaranteed, user-visible remediation hint, enforced by `test_require_cluster_deps_message`.
- **Import failure of `reasons.cluster` itself**: swallowed by the top-level `try/except ImportError`, which sets `HAS_CLUSTER_DEPS = False`. Most tests then skip. Note the side effect: the undecorated `_auto_k` tests would raise `NameError` (not skip) in this pathological case — arguably intended, since a fully broken import should be loud, not silent.
- **Test-time assertion failures**: no catching inside tests; plain `assert` statements propagate as pytest failures, as expected.

## Topics to Explore

- [file] `reasons/cluster.py` — The module under test; read `cluster_beliefs`, `list_clusters`, `ClusterCache`, `_auto_k`, `_require_cluster_deps` to see how the contracts are implemented (especially the under-budget short-circuit and the small-corpus guard).
- [function] `reasons/cluster.py:cluster_beliefs` — Focus on how budget is allocated across clusters (does it proportionally sample, round-robin, or cap per cluster?) and why cross-domain coverage holds.
- [function] `reasons/cluster.py:_auto_k` — The `max(2, n // 5)` / `max_k` / `n_clusters` clamp heuristic; small but central to reproducibility and cluster-count expectations.
- [file] `reasons/derive.py` — The consumer of this clustering (per the file docstring); see how `budget` and `stats` flow into the derive result and what happens if clustering is unavailable.
- [general] `optional-dependency-extras` — Inspect `pyproject.toml` for the `ftl-reasons[cluster]` extra definition and how `HAS_CLUSTER_DEPS` is computed inside `reasons/cluster.py` (lazy vs. eager imports).

## Beliefs

- `cluster-numeric-tests-run-without-extras` — The three `_auto_k` tests and `test_require_cluster_deps_message` are the only tests without `@skip_no_cluster`; they exercise pure arithmetic or the missing-dep path.
- `cluster-beliefs-returns-exactly-budget` — When `0 < budget < len(beliefs)`, `cluster_beliefs` returns exactly `budget` ids, not at most `budget`.
- `cluster-selection-preserves-cross-domain-diversity` — With a mixed auth/database corpus and `budget=20`, `cluster_beliefs` selects at least one belief from each domain.
- `cluster-stats-account-for-all-beliefs` — `sum(cluster_beliefs(...)[1]["cluster_sizes"])` equals the number of input beliefs, and stats include `n_clusters`, `cluster_sizes`, and `embedding_model`.
- `cluster-cache-is-incremental` — `ClusterCache.embed` grows `_cache` only by the number of previously unseen belief ids and returns unchanged `ids` on repeat calls.
- `list-clusters-partitions-all-beliefs` — The union of ids across `list_clusters(...)["clusters"]` equals the input key set, with no duplicates.
- `auto-k-is-n-over-5-capped` — `_auto_k(n)` defaults to `max(2, n // 5)`, `max_k` caps that default, and an explicit `n_clusters` overrides it but is clamped to `n`.
