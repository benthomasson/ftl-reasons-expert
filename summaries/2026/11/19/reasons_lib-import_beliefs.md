---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/import_beliefs.py

# `reasons_lib/import_beliefs.py`

## Purpose

This file is a **backwards-compatibility shim**. Its only job is to keep the legacy import path

```python
from reasons_lib.import_beliefs import ...
```

working while the canonical implementation lives at:

```python
from reasons.import_beliefs import ...
```

The docstring on line 1 explicitly says:

> Backwards-compatibility shim — use `from reasons.import_beliefs` instead.

So this file owns **no import/parsing/belief-loading logic of its own**. It delegates entirely to `reasons.import_beliefs`. The canonical module is the source of truth for behavior, symbols, and side effects.

## Key Components

The file defines no classes, functions, or constants. It has exactly two meaningful pieces:

1. **Module docstring** — declares the compatibility intent and points developers to the canonical module.
2. **Wildcard import**:

```python
from reasons.import_beliefs import *  # noqa: F401,F403
```

Contract:

- Any public name exported by `reasons.import_beliefs` becomes available as `reasons_lib.import_beliefs.<name>`.
- Existing code such as `from reasons_lib.import_beliefs import SomeSymbol` continues to work if `SomeSymbol` is public in the canonical module.
- Private names beginning with `_` are not exported by `import *` unless the canonical module explicitly includes them in `__all__`.
- The objects exposed here are the same Python objects as in `reasons.import_beliefs` at import time. For example, if the canonical module defines a function `foo`, then `reasons_lib.import_beliefs.foo is reasons.import_beliefs.foo` is true.

The `# noqa: F401,F403` comment suppresses linter warnings:

- `F401`: imported but unused.
- `F403`: star import used.

It has no runtime effect.

## Patterns

- **Compatibility shim / facade pattern**: the module adapts the old import path to the canonical one without duplicating implementation.
- **Namespace migration / dual-package layout**: the repository contains both `reasons/` and `reasons_lib/`; this file lets old `reasons_lib` imports survive after the project moved to `reasons`.
- **Delegation over duplication**: the shim intentionally contains no business logic.
- **Wildcard re-export**: Python’s `import *` copies public names from the canonical module into this module’s globals.
- **Lint suppression for intentional re-export**: `# noqa: F401,F403` marks the wildcard import as deliberate.

## Dependencies

This module depends on:

```python
reasons.import_beliefs
```

That dependency is evaluated at import time. Importing `reasons_lib.import_beliefs` first imports `reasons.import_beliefs` and then copies its public names.

According to the provided dependency map, this shim is imported by:

- `reasons/api.py`
- `reasons/import_agent.py`
- `reasons/pg.py`
- `tests/test_import_beliefs.py`
- `tests/test_metadata.py`
- `tests/test_metadata_extended.py`
- `tests/test_nogood_id.py`
- `tests/test_roundtrip.py`
- `tests/test_source_type.py`

A developer maintaining this should keep the canonical module as the primary target. If the canonical module changes its public API, this shim automatically reflects those changes on next import, provided the names are public and/or included in the canonical module’s `__all__`.

## Flow

Import execution is simple:

1. Python imports `reasons_lib.import_beliefs`.
2. The module docstring is evaluated, which is a no-op.
3. Python executes `from reasons.import_beliefs import *`.
4. Python imports `reasons.import_beliefs` if it is not already in `sys.modules`.
5. The canonical module is fully initialized, including any module-level side effects.
6. Python copies the exported names into the shim’s global namespace.
7. The shim module finishes loading and is cached in `sys.modules`.

Export rules are standard Python semantics:

- If `reasons.import_beliefs` defines `__all__`, the star import copies exactly those names.
- Otherwise, it copies all module-level names that do not begin with `_`.

The shim itself does **not** define `__all__`. Therefore, if a downstream module does:

```python
from reasons_lib.import_beliefs import *
```

Python exports the non-underscore names present in the shim’s globals, which are the names brought in by the canonical star import.

## Invariants

- `reasons.import_beliefs` is the canonical source of truth.
- `reasons_lib.import_beliefs` must not add independent business logic.
- The shim must remain importable for backwards compatibility.
- The canonical module must be importable; otherwise this shim cannot load.
- Private names from the canonical module are not guaranteed to be re-exported.
- Import-time side effects from `reasons.import_beliefs` occur as part of importing this shim.
- The shim has no lazy forwarding or `__getattr__`; missing attributes raise normal `AttributeError`.
- Rebinding a name inside `reasons.import_beliefs` after this shim is imported does not automatically update the shim’s already-bound global name.

There is also a circular-import caveat: because line 2 imports the canonical module immediately, if `reasons.import_beliefs` directly or indirectly imports `reasons_lib.import_beliefs` while the canonical module is still initializing, Python may expose a partially initialized module. The shim should remain a one-way compatibility layer, not part of the canonical module’s dependency cycle.

## Error Handling

This file performs no error handling:

- No `try`/`except`.
- No `ImportError` fallback.
- No validation of exported names.
- No logging.

Any failure inside `reasons.import_beliefs` propagates unchanged. For example:

- If `reasons.import_beliefs` is missing or has a broken dependency, importing this shim raises `ModuleNotFoundError` or `ImportError`.
- If the canonical module raises an exception during initialization, that exception propagates through this shim.
- If a consumer imports a name that the canonical module does not export, Python raises `ImportError` at the consumer’s import site.

The `# noqa: F401,F403` only affects static analysis; it does not swallow runtime errors.

## Topics to Explore

- [file] `reasons/import_beliefs.py` — The canonical implementation whose public API this shim re-exports.
- [file] `tests/test_import_beliefs.py` — Tests the import-beliefs behavior and backward-compatibility expectations.
- [file] `reasons_lib/__init__.py` — Shows how the legacy `reasons_lib` namespace is structured alongside the shim.
- [general] `star-import-semantics` — Python `__all__`, wildcard import, and re-export behavior are central to this file.
- [file] `reasons/api.py` — A production consumer listed as importing this compatibility path.

## Beliefs

- `reasons-lib-import-beliefs-is-a-compatibility-shim` — This file exists only to keep old `reasons_lib.import_beliefs` imports working.
- `reasons-lib-import-beliefs-requires-reasons-import-beliefs` — Importing this module requires the canonical `reasons.import_beliefs` module to be importable.
- `reasons-lib-import-beliefs-propagates-canonical-import-errors` — It has no exception handling, so canonical import failures propagate unchanged.
- `reasons-lib-import-beliefs-has-no-explicit-all` — The shim does not define `__all__`; its exported names come from the wildcard import on line 2.
- `reasons-lib-import-beliefs-star-imports-canonical-public-api` — The only runtime operation is `from reasons.import_beliefs import *`.
