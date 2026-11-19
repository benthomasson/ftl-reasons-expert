---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: How `ftl-reasons[cluster]` is declared in `pyproject.toml` and guarded across the project.

# `ftl-reasons[cluster]` — Declaration and Guarding Strategy

## 1. Declaration in `pyproject.toml`

The distribution name is `ftl-reasons`, declared on **line 5** of `pyproject.toml`:

```toml
name = "ftl-reasons"
```

The `[project.optional-dependencies]` table (line 13) exposes a **`cluster` extra** on **line 17**:

```toml
cluster = ["sentence-transformers>=2.2", "scikit-learn>=1.2"]
```

That is what `pip install 'ftl-reasons[cluster]'` resolves to. Two things worth noting:

- The extra is named `cluster`, and the canonical install string is derived from `[project].name` + `[project.optional-dependencies].cluster`.
- Other extras (`pg`, `mcp`, `api`, `vertex`, `langfuse`, `openai`, `llm`, `test`, `test-pg`) follow the same pattern, so the dependency-guard idiom is reused across the project.
- `dependencies = []` (line 11) — cluster is **strictly optional**; the base install must import `reasons.cluster` without any of these packages present.

## 2. Import-time guard in `reasons/cluster.py`

The module self-declares availability with a soft-import block at the top (roughly lines 12–19):

```python
try:
    from sentence_transformers import SentenceTransformer
    import numpy as np
    from sklearn.cluster import KMeans
    HAS_CLUSTER_DEPS = True
except ImportError:
    SentenceTransformer = None
    np = None
    KMeans = None
    HAS_CLUSTER_DEPS = False
```

This has two consequences:

1. **`import reasons.cluster` always succeeds** — importers never crash when the extra is missing.
2. `HAS_CLUSTER_DEPS` is the single source of truth tests and callers can branch on.

Note that `numpy` is grouped into this guard even though it isn't listed in the `cluster` extra in `pyproject.toml`; it's transitively pulled in by `scikit-learn`/`sentence-transformers`.

## 3. Runtime guard: `_require_cluster_deps`

Defined in `reasons/cluster.py` around lines 26–31:

```python
def _require_cluster_deps():
    if not HAS_CLUSTER_DEPS:
        raise ImportError(
            "sentence-transformers and scikit-learn are required for --cluster mode. "
            "Install them with: pip install 'ftl-reasons[cluster]'"
        )
```

The error string embeds the exact `pip install 'ftl-reasons[cluster]'` command — the same extra name declared in `pyproject.toml` — so the "how to fix" answer is always in the traceback.

Every public entry point in the module begins with `_require_cluster_deps()`:

| Function | Guard line |
|---|---|
| `ClusterCache.__init__` | calls `_require_cluster_deps()` |
| `cluster_beliefs` | `_require_cluster_deps()` first statement |
| `cluster_beliefs_intra` | `_require_cluster_deps()` first statement |
| `list_clusters` | `_require_cluster_deps()` first statement |

So the strategy is **"soft import, hard call"**: the symbol is always importable, but any attempt to actually use clustering fails loudly with a remediation hint.

## 4. Call-site guarding

Two patterns show up at call sites:

**Lazy, local imports.** `reasons/api.py` imports `reasons.cluster` **inside** `list_clusters` (line 4954) and `find_duplicates` (line 4998):

```python
from .cluster import list_clusters as _list_clusters, DEFAULT_MODEL
...
from .cluster import ClusterCache, DEFAULT_MODEL
```

Because the import is inside the function body, a missing extra never leaks into module-load time of `reasons.api`. The `ImportError` from `_require_cluster_deps()` propagates only when the user actually invokes the clustering path (e.g. `reasons/api.py:3599` where `cluster=True` triggers `list_clusters`).

**CLI catch.** `reasons/cli.py:1696` has `except ImportError as e:` around the clustering path, which is how the friendly message is surfaced to end users of the `reasons` script.

**Re-export shim.** `reasons_lib/cluster.py` is a single line:

```python
from reasons.cluster import *  # noqa: F401,F403
```

So `reasons_lib.cluster.HAS_CLUSTER_DEPS` and `_require_cluster_deps` are the *same objects* as in `reasons.cluster`; the guard doesn't need to be replicated in the legacy namespace.

## 5. Test-side guarding

`tests/test_cluster.py` mirrors the module guard at the top (lines 10–18):

```python
try:
    from reasons.cluster import (
        cluster_beliefs, list_clusters, ClusterCache, _require_cluster_deps,
        _auto_k, HAS_CLUSTER_DEPS,
    )
except ImportError:
    HAS_CLUSTER_DEPS = False

skip_no_cluster = pytest.mark.skipif(
    not HAS_CLUSTER_DEPS,
    reason="sentence-transformers and scikit-learn not installed",
)
```

Every heavy test is decorated with `@skip_no_cluster`. The missing-dep **error path** is separately tested by `test_require_cluster_deps_message` (lines 22–28):

```python
with pytest.raises(ImportError, match="ftl-reasons\\[cluster\\]"):
    _require_cluster_deps()
```

That test only runs when deps are *absent* — so the regex on the canonical install string is validated in CI jobs without the extra. Pure functions like `test_auto_k_defaults` / `test_auto_k_with_override` / `test_auto_k_with_max_k` are **not** decorated, because `_auto_k` has no dependency on `sentence-transformers`/`sklearn`.

The same `skip_no_cluster = pytest.mark.skipif(...)` idiom is replicated in the other test files that touch clustering:

- `tests/test_api.py:783–787`
- `tests/test_cli.py:817–821`
- `tests/test_contradictions.py:377–380`
- `tests/test_derive.py:1023–1026`

## Summary of the contract

| Layer | Guarantee |
|---|---|
| `pyproject.toml` | `ftl-reasons[cluster]` installs exactly `sentence-transformers>=2.2` and `scikit-learn>=1.2` |
| Module import | `reasons.cluster` always imports; `HAS_CLUSTER_DEPS` tells you if the extra is installed |
| Function call | `_require_cluster_deps()` raises `ImportError("… pip install 'ftl-reasons[cluster]'")` on every public entry point |
| `reasons.api` / `reasons.cli` | Lazy-import + `except ImportError` so the base install never breaks on this path |
| `reasons_lib` | Pure star re-export; no separate guard |
| Tests | `skipif(HAS_CLUSTER_DEPS)` for behavior tests; a dedicated test asserts the exact `pip install 'ftl-reasons[cluster]'` message when deps are missing |

### Gaps in the observations

- The exact line numbers within `reasons/cluster.py` for `_require_cluster_deps` and `ClusterCache.__init__` were inferred from the visual content; only the try/except lines (14–19) and the raise (line 28) are explicitly cited.
- Nothing in the observations shows a *runtime* gate on the `reasons` CLI parser (e.g. making `--cluster` fail at parse time); the CLI catch at `reasons/cli.py:1696` is only one side of that behavior.

## Topics to Explore

- [file] `reasons/cluster.py` — Full module; verify which functions call `_require_cluster_deps()` and check for any path that bypasses it
- [function] `reasons/cli.py:1696` — The `except ImportError` handler; how CLI surfaces the missing-extra error vs. how it groups other import failures
- [function] `tests/test_cluster.py:test_require_cluster_deps_message` — The only test that runs the missing-deps branch (via `match="ftl-reasons\\[cluster\\]"`); worth confirming it is actually exercised in a no-cluster CI job
- [general] `optional-dependency-guard-pattern` — Compare how `cluster`, `pg`, `mcp`, `api`, `llm` extras are guarded; `pg.py` and `mcp_client.py` follow a similar `try/except ImportError` + `raise ImportError` shape, which may want a shared helper
- [diff] `pyproject.toml` history for the `cluster` extra — When `scikit-learn` was added, when `numpy` was folded into the guard block, and whether the extra was widened for sentence-transformers 3.x

## Beliefs

- `cluster-extra-requires-sentence-transformers` — The `cluster` optional dependency in `pyproject.toml` requires `sentence-transformers>=2.2`.
- `cluster-extra-requires-scikit-learn` — The `cluster` optional dependency in `pyproject.toml` requires `scikit-learn>=1.2`.
- `reasons-cluster-import-is-soft` — `reasons.cluster` never raises on import when the extra is absent; the `try/except ImportError` block sets `HAS_CLUSTER_DEPS = False` and nulls the dependency bindings.
- `_require_cluster_deps-raises-with-install-hint` — All four public callables (`ClusterCache.__init__`, `cluster_beliefs`, `cluster_beliefs_intra`, `list_clusters`) call `_require_cluster_deps()`, which raises `ImportError` containing `pip install 'ftl-reasons[cluster]'`.
- `reasons-api-imports-cluster-lazily` — `reasons/api.py` imports `reasons.cluster` inside function bodies (`list_clusters` at line 4954, `find_duplicates` at line 4998), keeping the base install free of the extra.
- `reasons-lib-cluster-is-reexport` — `reasons_lib/cluster.py` is a single `from reasons.cluster import *` line, so its guard and `HAS_CLUSTER_DEPS` are identical objects to those in `reasons.cluster`.
- `cluster-tests-skip-when-deps-missing` — Every behavior test in `tests/test_cluster.py` is decorated with `@skip_no_cluster = pytest.mark.skipif(not HAS_CLUSTER_DEPS, ...)`; only the pure `_auto_k` tests run unconditionally.
