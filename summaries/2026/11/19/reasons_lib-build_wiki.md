---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/build_wiki.py

# `reasons_lib/build_wiki.py`

## Purpose

This file is a **backwards-compatibility shim**. The canonical implementation lives in `reasons/build_wiki.py`, but legacy callers may still import from the old path:

```python
from reasons_lib.build_wiki import ...
```

The shim keeps that old import path working by re-exporting everything public from the canonical module. It owns no business logic, state, or validation — its only responsibility is compatibility.

## Key Components

There are no local classes, functions, or constants.

- **Module docstring**  
  Explicitly states the migration guidance: use `from reasons.build_wiki` instead of this module.

- **`from reasons.build_wiki import *`**  
  This is the entire behavior of the file. It imports the public namespace of the canonical module into this shim.

- **`# noqa: F401,F403`**  
  Suppresses lint warnings for:
  - `F401`: imported but unused
  - `F403`: wildcard import

  The suppression is intentional: this module exists to re-export names, not to use them locally.

- **No local `__all__`**  
  The shim does not curate its exported API. Its namespace is determined by star-import semantics from `reasons.build_wiki`.

## Patterns

This uses the **compatibility shim / facade** pattern:

- The old module path remains valid.
- The real implementation is delegated to the new module path.
- Consumers do not need to change immediately.
- The shim does **not** alias module objects in `sys.modules`; it is a distinct module object whose namespace is populated from the canonical module.

It also relies on **wildcard re-export**, which is convenient but makes the exported API implicit. If `reasons.build_wiki` defines `__all__`, that controls what this shim imports. If not, all non-underscore names from the canonical module’s namespace are imported, including imported modules/types that happen to be public.

## Dependencies

### Depends on

- `reasons.build_wiki`

This is a hard dependency. The shim cannot work unless the canonical module exists and imports successfully.

### Imported by

According to the repository context:

- `reasons/api.py`
- `tests/test_build_wiki.py`

That means the compatibility path is still part of the active dependency graph, including at least one production module and one test module.

## Flow

Execution is very simple:

1. A caller imports `reasons_lib.build_wiki`.
2. Python executes the shim module body.
3. The shim imports `reasons.build_wiki`.
4. Any top-level code in `reasons/build_wiki.py` runs.
5. The star import copies the canonical module’s exported names into the shim’s namespace.
6. The caller receives the shim module.

After import:

- `reasons_lib.build_wiki.some_function` is the same function object as `reasons.build_wiki.some_function`.
- The function’s runtime globals still belong to `reasons.build_wiki`, not to the shim.
- `import reasons_lib.build_wiki as old` and `import reasons.build_wiki as new` produce two distinct module objects; this is not a direct `sys.modules` alias.

A subtle consequence: because the shim has no local `__all__`, downstream `from reasons_lib.build_wiki import *` is governed by the shim’s namespace, not by the canonical module’s `__all__` attribute directly. Direct attribute access to names imported into the shim still works, but star-import behavior through the shim can differ in edge cases such as underscore-prefixed names explicitly listed in the canonical module’s `__all__`.

## Invariants

- `reasons_lib.build_wiki` must remain importable for legacy callers.
- The canonical module `reasons.build_wiki` must exist and be importable.
- The shim must not contain independent logic that diverges from the canonical implementation.
- The exported API surface is implicitly controlled by `reasons/build_wiki.py`; adding or removing public names there changes what this shim exposes.
- Private names from the canonical module are not normally re-exported through the old path unless the canonical module’s `__all__` explicitly includes them.
- Because the shim has no local `__all__`, it does not fully reproduce the canonical module’s star-import contract for downstream wildcard imports.

## Error Handling

There is no local error handling.

- If `reasons.build_wiki` is missing or fails during import, the resulting `ImportError` / `ModuleNotFoundError` or other exception propagates unchanged.
- If a caller requests a name that was not re-exported, normal `ImportError` or `AttributeError` behavior applies.
- No fallback implementation exists in this shim.

## Topics to Explore

- [file] `reasons/build_wiki.py` — The canonical implementation re-exported by this shim; verify its public API, `__all__`, and import-time side effects.
- [file] `tests/test_build_wiki.py` — Shows the expected contract and whether tests exercise the legacy `reasons_lib` path or the canonical `reasons` path.
- [file] `reasons/api.py` — Listed as an importer of this shim; confirm whether that dependency is intentional or a migration leftover.
- [file] `reasons_lib/__init__.py` — Reveals whether shimming is module-by-module or handled systematically at the package level.
- [general] `package-compatibility-strategy` — Compare the duplicate `reasons/` and `reasons_lib/` trees and packaging configuration to understand the migration rules.

## Beliefs

- `reasons-lib-build-wiki-is-a-compatibility-shim` — The file contains only a module docstring and a wildcard import from `reasons.build_wiki`, with no local business logic.
- `reasons-lib-build-wiki-requires-reasons-build-wiki` — Importing `reasons_lib.build_wiki` requires `reasons.build_wiki` to exist and import successfully.
- `reasons-lib-build-wiki-export-surface-follows-star-import` — Its re-exported names are determined by star-import semantics from `reasons.build_wiki`, not by an explicit local `__all__`.
- `reasons-lib-build-wiki-propagates-import-errors` — It has no try/except, so canonical-module import failures propagate to callers unchanged.
- `reasons-api-imports-compat-path` — Repository context lists `reasons/api.py` as an importer of `reasons_lib.build_wiki`, so the compatibility path is still used internally.
