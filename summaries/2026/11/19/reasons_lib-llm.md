---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/llm.py

# `reasons_lib/llm.py`

## Purpose

This file is a **backwards-compatibility shim**. It owns no business logic, no LLM client implementation, and no configuration. Its only responsibility is to keep the legacy import path:

```python
from reasons_lib.llm import ...
```

working after the canonical implementation moved to:

```python
from reasons.llm import ...
```

The module docstring states this explicitly:

```python
"""Backwards-compatibility shim — use ``from reasons.llm`` instead."""
```

So `reasons_lib/llm.py` is a migration bridge: old callers can continue importing from `reasons_lib.llm`, while new code should import from `reasons.llm`.

## Key Components

There are only two meaningful lines:

```python
"""Backwards-compatibility shim — use ``from reasons.llm`` instead."""
from reasons.llm import *  # noqa: F401,F403
```

### Module docstring

Documents the intended migration direction: `reasons.llm` is canonical, `reasons_lib.llm` is legacy.

### Wildcard re-export

```python
from reasons.llm import *  # noqa: F401,F403
```

This binds all public names exported by `reasons.llm` into the `reasons_lib.llm` module namespace.

Contract implications:

- For any public name `X` imported by the star import, `reasons_lib.llm.X` refers to the same object as `reasons.llm.X`.
- Functions, classes, constants, and module-level objects are not copied or wrapped; they are shared references.
- The shim does not define local classes, functions, constants, or `__all__`.
- The `# noqa: F401,F403` suppresses lint warnings for unused imports (`F401`) and wildcard imports (`F403`), which are intentional here.

### `__all__` nuance

The shim does **not** explicitly define or forward `__all__`.

- If `reasons.llm` defines `__all__`, the names listed there are imported into this module.
- The shim itself will not normally expose `__all__` unless `reasons.llm.__all__` explicitly includes `"__all__"`.
- A downstream `from reasons_lib.llm import *` therefore exports all non-underscore names present in the shim’s namespace, not necessarily exactly the same `__all__` contract as `reasons.llm`.

This is usually acceptable for a compatibility shim, but it is a subtle difference from a perfect re-export.

## Patterns

### Backwards-compatibility shim / adapter

The file implements the common “old import path delegates to new import path” pattern. It avoids duplicating implementation while preserving API compatibility.

### Wildcard re-export

Using `import *` is normally discouraged because it makes dependencies implicit and can pollute namespaces. Here it is deliberate: the goal is to mirror the public surface of another module with minimal maintenance.

### Eager import

The delegation happens at module import time. There is no lazy `__getattr__`, no dynamic dispatch, and no conditional import logic. If `reasons.llm` is importable, the shim succeeds; otherwise it fails immediately.

## Dependencies

### Depends on

- `reasons.llm` — the canonical implementation module.

Specifically, the shim relies on `reasons.llm` being importable and exposing the public names expected by legacy callers.

### Imported by

According to the repository context, this module is imported by several modules in the `reasons` package, including:

- `reasons/api.py`
- `reasons/ask.py`
- `reasons/build_wiki.py`
- `reasons/cli.py`
- `reasons/contradictions.py`
- `reasons/pg.py`
- `reasons/propose_update.py`
- `reasons/repair.py`
- `reasons/repair_premises.py`
- `reasons/report_belief.py`
- `reasons/report_gated.py`
- `reasons/review.py`
- `reasons/review_justifications.py`
- `reasons/review_premises.py`
- `reasons/verify.py`

and by:

- `tests/test_llm.py`

This is worth noting: even though `reasons_lib.llm` is described as legacy, parts of the `reasons` package still import it according to the static import graph. That may indicate an incomplete migration, or it may reflect compatibility imports that have not yet been updated to `reasons.llm`.

## Flow

The control flow is trivial:

1. Python imports `reasons_lib.llm`.
2. Python executes `from reasons.llm import *`.
3. Python imports `reasons.llm`.
4. Python determines which names to export from `reasons.llm`:
   - If `reasons.llm.__all__` exists, those names are imported.
   - Otherwise, all names in `reasons.llm` not starting with `_` are imported.
5. Those names are bound into the `reasons_lib.llm` module namespace.
6. Callers access them as `reasons_lib.llm.<name>`.

There is no transformation, validation, wrapping, logging, or runtime behavior beyond the import.

## Invariants

- **Canonical implementation lives in `reasons.llm`.** This file must not contain implementation logic.
- **Public names must be exported by `reasons.llm`.** If a legacy caller expects `reasons_lib.llm.foo`, then `reasons.llm` must expose `foo` as a public name, or explicitly include it in `__all__`.
- **Underscore-prefixed names are not re-exported by the wildcard import** unless they appear in `reasons.llm.__all__`.
- **Imported objects are shared references.** `reasons_lib.llm.some_function is reasons.llm.some_function` should hold for any wildcard-exported function.
- **No local `__all__` is defined here**, so downstream wildcard imports from this shim use Python’s default “all non-underscore names” behavior.
- **No circular-import protection exists.** If `reasons.llm` ever imports `reasons_lib.llm` during initialization, this shim can participate in a circular import. The current design assumes that does not happen.

## Error Handling

There is no explicit error handling.

- If `reasons.llm` does not exist or fails to import, Python raises `ImportError` / `ModuleNotFoundError` from this shim.
- If `reasons.llm` imports successfully but is missing a name expected by a caller, accessing that missing attribute later raises `AttributeError`.
- Underscore-prefixed names are silently omitted by the wildcard import rather than causing an error.
- The `# noqa` comment suppresses lint errors only; it does not affect runtime behavior.

A developer maintaining this file should keep it intentionally minimal. If compatibility needs to be more precise, consider replacing the wildcard import with an explicit re-export list or forwarding `__all__` explicitly.

## Topics to Explore

- [file] `reasons/llm.py` — The canonical LLM implementation that this shim re-exports; inspect its public API and `__all__` behavior.
- [file] `tests/test_llm.py` — Likely validates both the canonical module and this compatibility shim; useful for understanding expected import parity.
- [general] `backwards-compatibility-shim` — Explore the tradeoffs of wildcard re-exports versus explicit re-exports, especially around `__all__` and type checking.
- [file] `reasons/api.py` — One of the `reasons` modules listed as importing `reasons_lib.llm`; check whether this is a legacy import that should migrate to `reasons.llm`.
- [general] `package-rename-migration` — Understand why both `reasons/` and `reasons_lib/` exist and which package is intended to be canonical.

## Beliefs

- `reasons-lib-llm-is-a-shim` — `reasons_lib/llm.py` contains no implementation logic and only re-exports names from `reasons.llm`.
- `reasons-lib-llm-requires-reasons-llm` — Importing `reasons_lib.llm` requires `reasons.llm` to be importable.
- `reasons-llm-owns-public-api` — The public API exposed through this shim is determined by `reasons.llm` and its `__all__` or public non-underscore names.
- `star-import-excludes-underscore-names` — Names beginning with `_` in `reasons.llm` are not re-exported by this shim unless explicitly included in `reasons.llm.__all__`.
- `shim-propagates-import-errors` — This module has no error handling, so failures in `reasons.llm` propagate as import exceptions.
