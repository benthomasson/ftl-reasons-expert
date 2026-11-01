---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/build_wiki.py

# `reasons_lib/build_wiki.py` — Backwards-Compatibility Shim

## 1. Purpose

This file exists **solely** to preserve a public import path that older code (and downstream consumers) may still reference. The real implementation lives in `reasons/build_wiki.py`. This shim lets code written against the legacy `reasons_lib` package namespace continue to work without modification.

In the broader project structure, every file under `reasons_lib/` mirrors a corresponding file under `reasons/`. This is a classic **package-rename migration pattern**: the project was reorganized from `reasons_lib` to `reasons`, and these shims keep the old import surface alive during (or indefinitely for) backward compatibility.

## 2. Key Components

There are **no original components** here. The entire file is:

```python
"""Backwards-compatibility shim — use ``from reasons.build_wiki`` instead."""
from reasons.build_wiki import *  # noqa: F401,F403
```

- **The star import** re-exports every public name (all identifiers not starting with `_`) defined in `reasons/build_wiki.py`. If that module defines an `__all__`, only those names are re-exported; otherwise, all module-level names that don't start with an underscore are available.
- **`# noqa: F401,F403`** silences `pyflakes`/`ruff` warnings for "imported but unused" (F401) and "star-import not allowed" (F403). This is intentional — the import *is* used, it's just used by *other* modules that import from this path.
- **The docstring** is the primary documentation artifact: it tells the reader to use the canonical `reasons.build_wiki` path.

## 3. Patterns

| Pattern | How it applies |
|---|---|
| **Facade / Shim** | Presents the same public API under a deprecated namespace. Zero logic of its own. |
| **Star re-export** | `import *` delegates name resolution entirely to the source module. Adding a new public function to `reasons/build_wiki.py` automatically makes it available here — no maintenance needed. |
| **Mirror package structure** | `reasons_lib/` is a 1:1 shadow of `reasons/`. Every module in `reasons_lib/` is presumably a similar one-liner shim, making the migration transparent. |
| **Deprecation via docstring** | No `warnings.warn()` or `DeprecationWarning` is emitted. The only signal is the docstring text. This is a "soft deprecation" — it works, but the documentation nudges you. |

## 4. Dependencies

**Upstream (this file depends on):**
- `reasons.build_wiki` — the actual implementation. If that module is renamed, moved, or its public API changes, this shim breaks silently (star import just exposes fewer/different names).

**Downstream (depends on this file):**
- `reasons/api.py` — likely does `from reasons_lib.build_wiki import ...` or `from reasons.build_wiki import ...`. Given it imports from `reasons_lib`, it's still using the legacy path.
- `tests/test_build_wiki.py` — test suite for the wiki-building functionality. If it imports from `reasons_lib.build_wiki`, it's exercising the shim path.

> **Key insight:** The fact that `reasons/api.py` imports from `reasons_lib` rather than `reasons` directly means the shim is still on a live code path, not just a vestige. Removing it would require updating `api.py`.

## 5. Flow

There is no control flow. Execution is:

1. Python imports `reasons_lib.build_wiki`.
2. The module body executes: `from reasons.build_wiki import *` triggers import of `reasons.build_wiki` (if not already in `sys.modules`), then copies its public namespace into `reasons_lib.build_wiki`'s namespace.
3. Any subsequent `from reasons_lib.build_wiki import build_wiki` (or whatever the exported symbols are) resolves to the function/class originally defined in `reasons/build_wiki.py`.

The data flow is identical to importing from `reasons.build_wiki` directly — the shim adds zero transformation.

## 6. Invariants

- **Namespace equivalence:** For any name `N` that is public in `reasons.build_wiki`, `reasons_lib.build_wiki.N is reasons.build_wiki.N` must hold. The star import guarantees identity (same object), not a copy.
- **No private leakage:** Names starting with `_` in `reasons.build_wiki` are *not* re-exported (Python's star-import rule), unless explicitly listed in `__all__`.
- **Single source of truth:** All logic lives in `reasons/build_wiki.py`. This file must never contain its own function definitions, class definitions, or mutable state.

## 7. Error Handling

- **Import-time failure:** If `reasons.build_wiki` doesn't exist, raises a circular import, or has a syntax error, the `import *` statement raises `ImportError` / `SyntaxError` / `CircularImportError` at the point where `reasons_lib.build_wiki` is first imported. There is no try/except, no fallback, no lazy loading.
- **Runtime errors:** Since no code executes beyond the import, there are no runtime errors to handle. All error behavior is inherited from `reasons.build_wiki`.
- **Silent degradation risk:** If a function is removed from `reasons.build_wiki`, the star import simply won't re-export it. Code doing `from reasons_lib.build_wiki import removed_func` will get a clear `ImportError`, but code doing `import reasons_lib.build_wiki as bw; bw.removed_func()` will get an `AttributeError` with no hint that the canonical path is `reasons.build_wiki`.

---

## Topics to Explore

- [file] `reasons/build_wiki.py` — The actual implementation this shim delegates to; understanding its public API tells you exactly what this shim re-exports
- [file] `reasons_lib/__init__.py` — Reveals whether the entire `reasons_lib` package is a shim or if some modules have diverged, and whether `__all__` is defined at the package level
- [file] `reasons/api.py` — Confirms how the shim is consumed in production code and whether the import could be migrated to `reasons.build_wiki` directly
- [file] `tests/test_build_wiki.py` — Shows which import path the tests use and whether they exercise the shim or the canonical module
- [general] `reasons-lib-migration` — The broader `reasons_lib` → `reasons` package rename: how many shim files exist, whether a deprecation timeline is documented in `CHANGELOG.md` or `pyproject.toml`, and whether `reasons_lib` is still listed as an installable entry point

## Beliefs

- `reasons-lib-build-wiki-is-a-shim-for-reasons-build-wiki` — `reasons_lib/build_wiki.py` contains zero original logic; every public name it exposes is the identical object (by `is`) defined in `reasons/build_wiki.py`
- `reasons-api-py-imports-via-reasons-lib` — `reasons/api.py` still imports wiki-building symbols through the `reasons_lib` namespace, meaning the shim is on a live production code path and cannot be deleted without updating `api.py`
- `star-import-guarantees-identity-not-copy` — Because the shim uses `import *` (not explicit rebinding), `reasons_lib.build_wiki.some_func is reasons.build_wiki.some_func` evaluates to `True`; monkey-patching the function in one module is visible through the other
- `no-deprecation-warning-emitted` — Importing from `reasons_lib.build_wiki` does not raise a `DeprecationWarning` or `PendingDeprecationWarning`; the only deprecation signal is the docstring text
- `reasons-lib-mirrors-reasons-one-to-one` — Every module in `reasons/` has a corresponding one-line shim in `reasons_lib/`, making the two packages structurally isomorphic and the shim layer uniformly thin
