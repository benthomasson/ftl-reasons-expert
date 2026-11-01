---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/export_card.py

# `reasons_lib/export_card.py`

## Purpose

This file is a **backwards-compatibility shim**. It exists solely to keep old import paths working after the project was renamed from `reasons_lib` to `reasons`. If any downstream code (or this repo's own `reasons/api.py` and `tests/test_export_card.py`) still does:

```python
from reasons_lib.export_card import export_card
```

…this file makes that resolve to the same symbols that now live in `reasons/export_card.py`. There is zero logic here — it's a one-line re-export with a star import.

## Key Components

There are no functions, classes, or constants defined in this file. The only "component" is the wildcard re-export:

```python
from reasons.export_card import *  # noqa: F401,F403
```

- **`*` import**: Re-exports every public name (anything not starting with `_` and not in `__all__`) from `reasons.export_card` into the `reasons_lib.export_card` namespace.
- **`# noqa: F401`**: Suppresses flake8's "imported but unused" warning — the import *is* used, just not locally.
- **`# noqa: F403`**: Suppresses the "star import" style warning, since this is intentional and scoped to a compat shim.

## Patterns

- **Shim / adapter for legacy import paths.** The `reasons_lib/` package mirrors the structure of `reasons/` one-to-one. Every file in `reasons_lib/` is presumably a thin `from reasons.X import *` re-export. This is the classic "rename a top-level package but don't break consumers" pattern.
- **Wildcard re-export as a compatibility contract.** The shim guarantees that the *set* of public symbols is identical between the two paths. If `reasons/export_card.py` adds a new public function, it automatically appears in `reasons_lib/export_card.py` without touching this file.

## Dependencies

| Direction | Target | Notes |
|-----------|--------|-------|
| **Imports** | `reasons.export_card` | The single real dependency; all symbols flow from there. |
| **Imported by** | `reasons/api.py` | The public API layer still references the old path (or this was a historical import that was left in place). |
| **Imported by** | `tests/test_export_card.py` | Tests exercise the `reasons_lib` path to ensure the shim works. |

## Flow

Execution is trivial:

1. Python loads `reasons_lib/export_card.py`.
2. The star import executes, resolving `reasons.export_card` and copying its public namespace into `reasons_lib.export_card.__dict__`.
3. Any subsequent `from reasons_lib.export_card import <name>` or `reasons_lib.export_card.<name>` accesses the same objects that `reasons.export_card` exposes.

There is no conditional logic, no state, no side effects beyond the namespace copy.

## Invariants

- **Namespace parity**: `dir(reasons_lib.export_card)` (excluding dunder attributes) must be a superset of `dir(reasons.export_card)`. Because it's a star import, this is guaranteed as long as `reasons.export_card` doesn't define `__all__` that excludes names.
- **No divergence**: This file must never define its own functions or variables. Any logic added here would silently shadow the real implementation and break the shim contract.
- **Removal safety**: Once `reasons/api.py` and `tests/test_export_card.py` are updated to import from `reasons.export_card` directly, this file (and the rest of `reasons_lib/`) can be deleted.

## Error Handling

There is none in this file. Two relevant cases:

- **Import-time failure**: If `reasons.export_card` itself fails to import (missing dependency, syntax error), the star import raises `ImportError` here. The error message will point at `reasons_lib/export_card.py` as the import site, which can be confusing — the real failure is one level down.
- **Name resolution at use-time**: If a consumer requests a name that `reasons.export_card` doesn't export (e.g., a private `_helper`), they get a standard `ImportError: cannot import name 'X' from 'reasons_lib.export_card'`.

No exceptions are caught, logged, or re-raised. It's a pure passthrough.

---

## Topics to Explore

- [file] `reasons/export_card.py` — The actual implementation this shim delegates to; understanding its public API (function signatures, return types) is the whole point of reading the shim.
- [file] `reasons/api.py` — Why does the public API module still import from the `reasons_lib` path? Tracing this reveals whether the migration to `reasons.*` is complete or intentionally partial.
- [general] `reasons_lib-migration` — The broader `reasons_lib/` → `reasons/` rename. Every file in `reasons_lib/` is likely an identical shim; understanding the migration timeline clarifies when (and whether) the old package can be removed.
- [file] `tests/test_export_card.py` — The test file that imports via `reasons_lib.export_card` confirms the shim is exercised in CI, but also shows what the public contract of `export_card` looks like from the consumer's perspective.
- [repo] `ftl-reasons/pyproject.toml` — Check whether both `reasons` and `reasons_lib` are registered as top-level packages (e.g., via `[tool.setuptools.packages]` or a flat layout). This determines whether the shim is needed for external consumers or only for in-repo imports.

---

## Beliefs

- `reasons_lib-export-card-is-a-reasons-export-card` — `reasons_lib/export_card.py` is a pure re-export shim; it defines no functions, classes, or module-level state of its own.
- `reasons-lib-shim-guarantees-namespace-parity` — Any public name added to `reasons/export_card.py` automatically becomes available via `reasons_lib.export_card` without modifying the shim, because the star import is resolved at import time.
- `reasons-lib-removal-blocked-by-api-and-tests` — The `reasons_lib/` package cannot be deleted until both `reasons/api.py` and `tests/test_export_card.py` are updated to import from `reasons.*` directly.
- `shim-star-import-ignores-__all__` — If `reasons/export_card.py` later defines an `__all__` list, the shim's star import will respect it (importing only the listed names), potentially breaking consumers who rely on unlisted public symbols.
- `reasons-lib-mirrors-reasons-one-to-one` — The directory structure of `reasons_lib/` is a 1:1 copy of `reasons/`, indicating a bulk package rename rather than a selective migration.
