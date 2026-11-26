---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Function: _require_cluster_deps in reasons/cluster.py

## `_require_cluster_deps`

### Purpose

`_require_cluster_deps` is a **capability guard** for the optional `--cluster` mode. It exists so that `reasons/cluster.py` can be imported safely in any installation, while still failing fast and loudly the moment the user tries to actually use clustering without having installed the optional extras.

The module intentionally uses a soft-import pattern:

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

The names are bound to `None` as a placeholder so that module-level references (type hints, `is None` checks) still work. `_require_cluster_deps` is the single place where that latent failure is converted into a real `ImportError` with an actionable message.

### Contract

- **Precondition:** none — the function is safe to call at any time.
- **Postcondition (success):** returns `None` and guarantees that `HAS_CLUSTER_DEPS` is `True`. Because `HAS_CLUSTER_DEPS` is computed at import time, success means the deps were available *when the module was imported*, not necessarily at call time.
- **Postcondition (failure):** raises `ImportError` with a user-facing message containing the pip install command.
- **Invariant:** callers must assume that *if* this function returned, then `SentenceTransformer`, `np`, and `KMeans` are all bound to real objects — not to `None`.

### Parameters

None. The function is parameterless — the state it inspects (`HAS_CLUSTER_DEPS`) is a module-level global.

### Return Value

`None` on success. The contract is entirely in the raise-or-return behavior, so callers should not depend on a truthy return value.

### Algorithm

1. Read the module-level `HAS_CLUSTER_DEPS` flag.
2. If `False`, raise `ImportError` with a message listing the missing packages and the exact pip install command (`pip install 'ftl-reasons[cluster]'`).
3. If `True`, do nothing.

There is no re-import, no `importlib.reload`, and no introspection at call time — this is a pure check of the import-time flag.

### Side Effects

- **Success:** none. No imports, no I/O, no state changes.
- **Failure:** raises an `ImportError`. Since it's typically called at the top of public API functions, this becomes the observable behavior of those functions.

### Error Handling

- **Raises `ImportError`** only — never a custom exception type. This means callers that already have a `try/except ImportError` (e.g., a CLI wrapper that wants to print a friendly message) will catch it.
- The check is **aggregate**: because the try-block imports `sentence_transformers`, `numpy`, and `sklearn` in one shot, a missing *any one* of the three fires the same error. The message names only `sentence-transformers` and `scikit-learn`; `numpy` is an implicit requirement that never gets mentioned.
- Stale-state risk: if the deps were uninstalled after module import (or in some exotic in-process environment), `HAS_CLUSTER_DEPS` will still be `True` and subsequent uses of `SentenceTransformer`, `np`, `KMeans` will raise their own errors.

### Usage Patterns

Every public entry point in `reasons/cluster.py` calls it as the *first statement*, before any shortcut logic:

```python
def cluster_beliefs(...):
    _require_cluster_deps()
    if len(beliefs) <= budget:   # shortcut path STILL requires deps
        return ...
```

This is an important caller-visible detail: even the trivial "no clustering needed" path (`len(beliefs) <= budget`, or `len(beliefs) <= 3` in `list_clusters`) will raise `ImportError` if the cluster extra is not installed. That sequencing is deliberate — it keeps behavior predictable across all inputs — but it means a user cannot use `list_clusters`/`cluster_beliefs` as a pass-through utility without installing extras.

`ClusterCache.__init__` also calls it, so the check is effectively duplicated when `cache=None` is passed into any of the public functions (once at the entry point, once inside the constructor).

**Caller obligations:**
- If you catch `ImportError` here, you own rendering the "install the extra" UX — this function only provides the raw message.
- You may not skip calling it and assume the module-level names are real; a `None` `SentenceTransformer` will fail opaquely later.
- You should not cache the result — call it each time you enter the module's public API to future-proof against reload scenarios.

### Dependencies

- **Internal:** the module-level `HAS_CLUSTER_DEPS` global defined in the `try/except ImportError` block.
- **External (transitively):** `sentence_transformers`, `numpy`, `sklearn.cluster.KMeans`. Not imported by this function itself.

### Assumptions Not Enforced by Types

1. `HAS_CLUSTER_DEPS` is a `bool` — nothing stops a caller from assigning to it, but the try-block makes this safe in practice.
2. The pip package is named `ftl-reasons` with a `[cluster]` extra — if the packaging config ever renames the extra, the error message becomes stale with no compile-time signal.
3. The three imports are co-needed. If the module ever adds a fourth optional dep (say `umap`), it must be folded into the same `try` block or the guard becomes incomplete.
4. `numpy` is functionally a required dep but is not named in the message — a common source of user confusion when only numpy is missing.

---

## Topics to Explore

- [file] `reasons/cluster.py` — See how `ClusterCache.embed` and the three public functions consume the guarded imports; especially the sha256-keyed cache in `ClusterCache`.
- [function] `reasons/cluster.py:ClusterCache` — The embedding cache uses `(node_id, sha256(text)[:16])` as its key, so text changes invalidate cache entries even for the same node.
- [file] `pyproject.toml` — Confirm the `[cluster]` extra actually lists `sentence-transformers`, `scikit-learn`, and `numpy`; the error message's pip command depends on this.
- [general] `optional-dependency-pattern` — Compare the `HAS_*_DEPS` soft-import pattern here with the rest of `ftl-reasons` to see if it's used consistently across optional subsystems.
- [file] `tests/test_cluster.py` — Discover how tests simulate `HAS_CLUSTER_DEPS=False` (likely by monkeypatching) to exercise the `ImportError` branch without uninstalling deps.

---

## Beliefs

- `cluster-deps-detected-at-import-time` — `HAS_CLUSTER_DEPS` is determined once when `reasons.cluster` is imported, not on every call to `_require_cluster_deps`.
- `cluster-deps-raise-importerror` — The function raises stdlib `ImportError`, never a custom exception type.
- `cluster-deps-check-precedes-shortcuts` — `cluster_beliefs` and `list_clusters` call `_require_cluster_deps()` before their early-return shortcuts, so even trivial inputs require the cluster extra.
- `cluster-deps-aggregate-missing` — A single `HAS_CLUSTER_DEPS` flag gates all three imports (`sentence_transformers`, `numpy`, `sklearn`), so any one missing produces the same message.
- `error-message-omits-numpy` — The raised `ImportError` text names only `sentence-transformers` and `scikit-learn`, but `numpy` is also required and will fire the same error if missing.
