---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/export_markdown.py

# `reasons_lib/export_markdown.py`

## 1. Purpose

This file is a **pure backwards-compatibility shim**. It exists solely so that any existing code doing `from reasons_lib.export_markdown import ...` continues to work after the canonical implementation was moved to `reasons.export_markdown`. The docstring makes the intent explicit:

> *"Backwards-compatibility shim — use `from reasons.export_markdown` instead."*

It owns **no logic, no state, and no public API of its own**. Its only responsibility is to re-export the entire public namespace of `reasons.export_markdown` under the old `reasons_lib` package name. In a monorepo where `reasons/` is the new canonical package and `reasons_lib/` is the legacy one, this file is the glue that prevents breaking changes for downstream importers.

## 2. Key Components

There are no classes, functions, or constants defined here. The entire module body is a single statement:

```python
from reasons.export_markdown import *  # noqa: F401,F403
```

- **`import *`** — pulls every name not prefixed with `_` from `reasons.export_markdown` into this module's namespace. After this line, `reasons_lib.export_markdown.X` is the same object as `reasons.export_markdown.X` for every public `X`.
- **`# noqa: F401,F403`** — suppresses two flake8/pyflakes warnings:
  - **F401** ("module imported but unused") — the star-import *is* the point, but the linter can't see that.
  - **F403** ("star-import makes names unavailable for static lookup") — the linter can't verify what names exist, so it warns.

The docstring on line 1 is the only "documentation" a consumer of this module will see via `help()`.

## 3. Patterns

- **Facade / Aliasing Module**: The file is a textbook example of the *module alias* pattern used during package renames. The new package (`reasons`) is the source of truth; the old package (`reasons_lib`) is a thin re-export layer. This lets a project rename its top-level package across release cycles without a hard breaking cut.
- **Star-import shim**: Rather than listing every public symbol (`from reasons.export_markdown import export_markdown, render_belief, ...`), the wildcard import guarantees that *any* new symbol added to `reasons.export_markdown` is automatically visible through the shim without touching this file. The trade-off is loss of static analyzability, which the `noqa` comments explicitly acknowledge.
- **Parallel package tree**: The repo contains two mirror trees (`reasons/` and `reasons_lib/`) with identical file names. This is a common pattern in Python projects migrating from a `_lib`-suffixed internal package to a cleaner public name.

## 4. Dependencies

### This file depends on (imports)
| Module | Relationship |
|---|---|
| `reasons.export_markdown` | The single upstream dependency. Every public name is re-exported from there. |

### This file is depended on (imported by)
| Importer | Notes |
|---|---|
| `reasons/api.py` | Notable: the *new* package's API module still imports through the old shim path, suggesting the migration is in-progress or `reasons.api` was written before the rename and hasn't been updated. |
| `tests/test_export_markdown.py` | Primary test for the export functionality; imports via `reasons_lib` for backward-compat coverage. |
| `tests/test_metadata.py` | Exercises metadata fields that appear in markdown exports. |
| `tests/test_metadata_extended.py` | Extended metadata tests, same surface. |
| `tests/test_roundtrip.py` | Round-trip (import → export) tests that serialize beliefs to markdown and back. |
| `tests/test_source_type.py` | Tests source-type annotations in exported markdown. |

> **Maintenance note**: The fact that `reasons/api.py` imports from `reasons_lib.export_markdown` (the shim) rather than directly from `reasons.export_markdown` is a small circularity that should be cleaned up. Once the `reasons_lib` package is fully deprecated, this import in `reasons/api.py` will break or become an unnecessary indirection.

## 5. Flow

Execution is trivially linear:

```
Interpreter imports `reasons_lib.export_markdown`
  → Python executes the single `from reasons.export_markdown import *`
    → Python imports `reasons.export_markdown` (if not already in `sys.modules`)
      → That module runs its own top-level code (defining functions, classes, etc.)
    → Python copies all public names into `reasons_lib.export_markdown`'s namespace
  → Importer receives a module object whose `__dict__` is a superset of the upstream
```

There is no conditional logic, no lazy loading, no `__getattr__` protocol. The namespace is fully populated at import time. After that, attribute lookups on `reasons_lib.export_markdown` are direct dictionary hits — no delegation, no proxy.

## 6. Invariants

- **Namespace parity**: At any point after a successful import, `set(dir(reasons_lib.export_markdown)) ⊇ set(dir(reasons.export_markdown))` (minus dunder names). The shim can only *add* names (it adds `__name__`, `__package__`, `__doc__`, etc.), never remove them.
- **Single source of truth**: No function or class is *defined* in this file. All behavior lives in `reasons.export_markdown`. Editing this file's re-export list (currently `*`) is the only way to change what's visible.
- **Import ordering**: `reasons.export_markdown` must be importable before this module can be imported. If `reasons` is not installed or the package is broken, this shim raises `ModuleNotFoundError` at import time with no fallback.

## 7. Error Handling

There is **no explicit error handling** in this file. Errors propagate naturally:

- If `reasons.export_markdown` doesn't exist, the `from ... import *` raises `ModuleNotFoundError` (or `ImportError`). The traceback points at this shim file, which can be confusing because the actual problem is in the upstream module.
- If `reasons.export_markdown` has a name that conflicts with a built-in, Python's import machinery resolves it (the imported name shadows the built-in in this module's namespace only). No explicit check is performed.
- There is no `try/except`, no `__getattr__` fallback, no lazy-import guard.

---

## Topics to Explore

- [file] `reasons/export_markdown.py` — The canonical implementation whose public API this shim re-exports; understanding its functions (e.g., `export_markdown`, `render_belief`) is prerequisite to understanding what consumers actually get.
- [file] `reasons_lib/__init__.py` — Check whether the entire `reasons_lib` package follows the same shim pattern for every module, or whether only some files are re-exports and others still contain real logic (a sign of partial migration).
- [file] `reasons/api.py` — This file imports from `reasons_lib.export_markdown` (the shim). Tracing that import reveals whether the migration from `reasons_lib` → `reasons` is complete or still in flight.
- [repo] `reasons/` vs `reasons_lib/` — Diff the two directory trees to confirm they are structurally identical and to identify any module that exists in one but not the other, which would indicate an incomplete rename.
- [general] `deprecation-lifecycle` — Understand the project's stated plan for removing `reasons_lib/` entirely (check `CHANGELOG.md`, `README.md`, and any deprecation warnings) so you know whether this shim is a permanent alias or a temporary bridge.

## Beliefs

- `reasons_lib-export_markdown-is-a-reason-export_markdown-alias` — Every public name accessible via `reasons_lib.export_markdown` is the identical Python object as the corresponding name in `reasons.export_markdown`; no wrapping, proxying, or transformation occurs.
- `reasons-api-imports-via-reasons_lib-shim` — `reasons/api.py` currently imports markdown-export symbols through `reasons_lib.export_markdown` rather than directly from `reasons.export_markdown`, meaning the new package still depends on the old shim path.
- `shim-star-import-guarantees-forward-compat` — Because the shim uses `import *` (not an explicit name list), any new public symbol added to `reasons.export_markdown` is automatically visible through `reasons_lib.export_markdown` without modifying the shim file.
- `shim-has-no-lazy-fallback` — If `reasons.export_markdown` is missing or broken, importing `reasons_lib.export_markdown` raises immediately at module-load time; there is no `__getattr__`, `try/except`, or deferred-import mechanism to defer or mask the failure.
- `reasons_lib-tree-mirrors-reasons-tree` — The `reasons_lib/` package contains a file-for-file mirror of the `reasons/` package (identical set of `.py` filenames), confirming that `reasons_lib` is a wholesale compatibility alias rather than a partial one.
