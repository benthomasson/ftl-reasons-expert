---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Whether the star import leaks incidental names (imports like `typing`, `dataclasses`) into the legacy namespace, and whether defining `__all__` upstream would tighten the contract

## Answer

Yes, with an important direction correction: the leak is from `reasons` into `reasons_lib`, not into the legacy `reasons` namespace. The star import does not add names to `reasons`; it copies names out of `reasons`.

`reasons_lib/__init__.py:1` does:

```python
from reasons import *  # noqa: F401,F403
```

Since `reasons/__init__.py` does **not** define `__all__` (`find_all_exports_definitions` found zero matches), Python falls back to importing every non-underscore global from `reasons`. `reasons/__init__.py:2` has:

```python
from dataclasses import dataclass, field
```

So `dataclass` and `field` are present in the `reasons` package namespace and are therefore re-exported into `reasons_lib` by the star import. To be precise: the `dataclasses` module itself is not leaked, but the incidental names `dataclass` and `field` are. `typing` is not present in the observed `reasons/__init__.py`.

The same pattern appears for the API wrapper: `reasons_lib/api.py:1` does `from reasons.api import *`, and `reasons/api.py` has no `__all__`. Its top-level import block includes non-underscore names such as `copy`, `json`, `logging`, `os`, `re`, `sqlite3`, `sys`, `Callable`, `datetime`, `timezone`, `combinations`, `Path`, plus internal names like `pubsub`, `build_meta`, `Network`, `Storage`, and `PROPOSALS_SCHEMA`. Those are candidates for leakage into `reasons_lib.api` via the star import.

The zero-result search for direct incidental stdlib imports in `reasons_lib` does not disprove this: it only shows `reasons_lib` does not directly import those modules. The names can still arrive indirectly through `from reasons import *` or `from reasons.api import *`.

Defining `__all__` upstream would tighten the contract, but only if placed in the **source module being star-imported**:

- In `reasons/__init__.py`, something like `__all__ = ["Justification", "Node", "Nogood"]` would make `from reasons import *` import only those names, not `dataclass` or `field`.
- In `reasons/api.py`, an `__all__` listing the intended public API would prevent implementation imports from being re-exported by `from reasons.api import *`.

Defining `__all__` only in `reasons_lib/__init__.py` would control downstream `from reasons_lib import *`, but it would not remove `reasons_lib.dataclass` or `reasons_lib.field` from the module namespace. The leak would remain accessible as attributes; only star-import exports from `reasons_lib` would be narrowed.

The observations are sufficient for the init-level conclusion. A runtime `dir(reasons_lib)` check is not shown, and `reasons/api.py` is only returned through line 300, but the star-import behavior and the observed absence of `__all__` are enough to establish the leak for the shown names.

## Topics to Explore

- [file] `reasons/__init__.py` — Add or evaluate `__all__ = ["Justification", "Node", "Nogood"]` to stop leaking `dataclass` and `field`.
- [file] `reasons_lib/__init__.py` — Replace the star import with explicit imports if you want to prevent incidental names from entering the new package namespace.
- [file] `reasons/api.py` — Identify the intended public API surface and define `__all__` to prevent `json`, `sqlite3`, `Path`, `Storage`, etc. from being star-exported.
- [file] `reasons_lib/api.py` — Check whether this wrapper should use explicit imports instead of `from reasons.api import *`.
- [general] `star-import-compatibility` — Search downstream callers before adding `__all__`, since code may be relying on incidental names like `from reasons import dataclass`.

## Beliefs

- `reasons-package-lacks-all` — The repository search found no `__all__` definitions, so `from reasons import *` exports all non-underscore globals from `reasons/__init__.py`.
- `reasons-init-binds-dataclass-and-field` — `reasons/__init__.py:2` imports `dataclass` and `field`, making those incidental names part of the `reasons` package namespace.
- `reasons-lib-init-star-imports-reasons` — `reasons_lib/__init__.py:1` imports all non-underscore names from `reasons`, including incidental `dataclass` and `field`.
- `reasons-api-star-import-leaks-implementation-names` — `reasons_lib/api.py:1` star-imports from `reasons/api.py`, which lacks `__all__` and exposes top-level names like `json`, `sqlite3`, `Path`, `Network`, and `Storage`.
- `all-in-source-limits-star-export` — Defining `__all__` in the source module limits what downstream star imports receive, but it does not hide those names from direct attribute access.
