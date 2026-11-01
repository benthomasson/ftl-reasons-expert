---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/compact.py

# `reasons_lib/compact.py`

## Purpose

This file is a **backwards-compatibility shim**. It exists solely so that code importing from the legacy `reasons_lib` package namespace continues to work after the real implementation was moved (or was always intended to live) in the `reasons` package. It owns no logic of its own — it re-exports the entire public surface of `reasons.compact` so that `from reasons_lib.compact import X` is semantically identical to `from reasons.compact import X`.

The docstring on line 1 makes this explicit:

```python
"""Backwards-compatibility shim — use ``from reasons.compact`` instead."""
```

This is a deprecation nudge: it works, but the project wants consumers to migrate their imports.

## Key Components

There is exactly **one statement** in this file:

```python
from reasons.compact import *  # noqa: F401,F403
```

| Element | Detail |
|---|---|
| `import *` | Wildcard-imports every name in `reasons.compact`'s public namespace (everything not starting with `_`, or explicitly listed in `__all__`). |
| `# noqa: F401` | Suppresses pyflakes' *"imported but unused"* warning. |
| `# noqa: F403` | Suppresses pyflakes' *"wildcard import — can't detect undefined names"* warning. |

No classes, functions, constants, or logic are defined here. The entire API surface is whatever `reasons.compact` exposes.

## Patterns

- **Mirror-package / shim pattern.** The `reasons_lib/` directory is a 1:1 structural mirror of `reasons/`. Every module in `reasons/` has a corresponding one-line file in `reasons_lib/` that does a star re-export. This is the standard Python idiom for renaming a top-level package while preserving backward compatibility for external consumers.
- **Delegation, not duplication.** No code is duplicated. If you change `reasons/compact.py`, the shim picks up the change automatically because it's a live import, not a copy.
- **No `__all__` defined locally.** The shim inherits whatever `__all__` (or the default "all public names") the target module defines.

## Dependencies

### What this file depends on
- **`reasons.compact`** (the real implementation). This is the *only* runtime dependency. If that module raises at import time (e.g., a missing optional dependency), this shim will raise the same error.

### What depends on this file
- **`reasons/api.py`** — The main API module imports from `reasons_lib.compact`. This is notable: even the "new" package's own `api.py` still routes through the shim, suggesting the migration is incomplete or that `reasons/api.py` predates the split.
- **`tests/test_compact.py`** — The test suite for the compact feature.
- **`tests/test_summarize.py`** — A test that exercises summarization logic that overlaps with compact.

## Flow

The control flow is trivially short:

1. Python's import machinery resolves `reasons_lib.compact`.
2. It executes the single line `from reasons.compact import *`.
3. Python's import machinery resolves `reasons.compact` (the real module) and runs its top-level code.
4. The wildcard import copies every public name from `reasons.compact`'s namespace into `reasons_lib.compact`'s namespace.
5. The importing module (`reasons/api.py`, a test file, etc.) binds those names locally.

There is no branching, no state mutation, no data transformation. It is a pure aliasing operation at import time.

## Invariants

- **Structural parity.** For every module `reasons/<name>.py`, a corresponding `reasons_lib/<name>.py` shim must exist. Adding a new module to `reasons/` without a matching shim in `reasons_lib/` will break any legacy consumer that imports from `reasons_lib`.
- **No local definitions.** This file must never gain its own functions, classes, or module-level state. Doing so would shadow the re-exported names and create a fork of behavior.
- **Import-order safety.** Because this is a star import, the set of re-exported names is determined entirely by `reasons.compact`. If `reasons.compact` adds or removes a public symbol, the shim's surface changes automatically — no edit to this file is needed, but also no safety net exists to catch accidental removals.

## Error Handling

There is **no explicit error handling** in this file. Errors propagate directly:

- If `reasons.compact` fails to import (syntax error, missing dependency like `networkx` or `sqlalchemy`), the `ImportError` / `ModuleNotFoundError` propagates to whoever imported `reasons_lib.compact`.
- If `reasons.compact` defines an `__all__` that references a name that doesn't exist, the wildcard import will raise `ImportError` at this line.
- No try/except, no fallback, no warning log. The shim is deliberately transparent — it should not mask or alter error semantics from the real module.

## Topics to Explore

- [file] `reasons/compact.py` — The actual implementation that this shim delegates to; understanding its public API (functions, classes, `__all__`) tells you exactly what surface the shim exposes.
- [file] `reasons/api.py` — The primary consumer that still imports through `reasons_lib.compact`; tracing this import reveals how far the migration from `reasons_lib` → `reasons` has progressed.
- [file] `tests/test_compact.py` — The test suite that exercises the compact feature; shows which names from `reasons.compact` are actually used in practice.
- [general] `reasons-lib-migration` — The broader pattern of the `reasons_lib/` → `reasons/` package rename; understanding why two parallel trees exist and when the shim layer is slated for removal.
- [file] `reasons_lib/__init__.py` — The package-level init of the shim layer; may contain additional re-exports or a deprecation warning that applies to all submodules.

## Beliefs

- `shim-delegates-to-real-module` — `reasons_lib/compact.py` defines zero logic; every public symbol it exposes is a live binding to a symbol in `reasons.compact`, so removing the shim without updating importers breaks `reasons/api.py` and the two test files that depend on it.
- `wildcard-import-inherits-all` — The set of names available via `from reasons_lib.compact import X` is determined entirely by `reasons.compact`'s public namespace at import time; adding or removing a public symbol in `reasons.compact` automatically changes what the shim exposes, with no edit to the shim file required.
- `noa-suppresses-both-f401-and-f403` — The `# noqa` comment on line 2 suppresses both F401 (unused import) and F403 (wildcard import) pyflakes codes, meaning linters will not flag this file; a maintainer should not "fix" these warnings by adding explicit imports, as that would defeat the purpose of the star re-export.
- `reasons-api-still-routes-through-shim` — `reasons/api.py` imports from `reasons_lib.compact` rather than `reasons.compact`, indicating the internal codebase has not yet completed its own migration off the legacy namespace.
