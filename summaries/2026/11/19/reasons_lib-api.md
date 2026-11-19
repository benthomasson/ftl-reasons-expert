---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/api.py

## Purpose

`reasons_lib/api.py` is a **backwards-compatibility shim**. It contains no business logic, classes, or functions of its own. Its only job is to keep the old import path:

```python
from reasons_lib.api import ...
```

working after the canonical API moved to:

```python
from reasons.api import ...
```

The module docstring states this explicitly: use `from reasons.api` instead. This file exists to avoid breaking existing code and tests while the project migrates from the `reasons_lib` namespace to the `reasons` namespace.

---

## Key Components

- **Module docstring, line 1**  
  Declares the file’s intent: it is a backwards-compatibility shim and the preferred import path is `reasons.api`.

- **`from reasons.api import *`, line 2**  
  The only executable statement. On import, Python loads `reasons.api` and copies its public export set into the `reasons_lib.api` module namespace.
  - If `reasons.api` defines `__all__`, only those names are re-exported.
  - Otherwise, Python imports all names from `reasons.api` that do not start with `_`.
  - The imported names become module globals of `reasons_lib.api`.

- **`# noqa: F401,F403`**  
  Suppresses lint warnings:
  - `F401`: imported but unused.
  - `F403`: star import used.
  
  This is intentional because the entire point of the import is to re-export names rather than use them locally.

- **Not present**
  - No local `__all__`.
  - No `__getattr__` dynamic forwarding.
  - No deprecation warning.
  - No try/except fallback.
  - No local classes, functions, or constants.

---

## Patterns

- **Compatibility shim / alias module**  
  This is a classic migration pattern: keep the old module path as a thin re-export layer over the new canonical module.

- **Star re-export**  
  The module uses `from reasons.api import *` rather than explicitly listing every symbol. This means its exported surface automatically follows `reasons.api`’s public export rules at import time.

- **Static re-export, not a proxy**  
  This is not a dynamic forwarding module. The names are bound into `reasons_lib.api` when the shim is imported. If `reasons.api` later rebinds a name, the shim’s existing binding will not automatically update.

- **Dual namespace migration**  
  The repository contains both `reasons/` and `reasons_lib/`. This file lets consumers continue using the legacy `reasons_lib.api` path while the canonical implementation lives under `reasons.api`.

- **Lint suppression for intentional re-export**  
  The `# noqa` comment is part of the design: linters would otherwise complain about an unused star import, but here the import is the module’s entire purpose.

---

## Dependencies

- **Depends on**
  - `reasons.api`, directly via `from reasons.api import *`.
  - Transitively, anything that `reasons.api` imports at module load time.

- **Imported by**
  - Canonical modules such as `reasons/ask.py`, `reasons/cli.py`, `reasons/contradictions.py`, `reasons/derive.py`, `reasons/mcp_server.py`, `reasons/pg.py`, and `reasons/repair.py`.
  - Many tests, including `tests/test_api.py`, `tests/test_derive.py`, `tests/test_verify.py`, etc.

This is important: the shim is not only used by external legacy consumers. Several modules inside the canonical `reasons` package still import from `reasons_lib.api`, so this compatibility layer participates in internal runtime imports too.

Potential dependency concern: the graph is:

```text
reasons_lib.api -> reasons.api
reasons.ask / reasons.cli / ... -> reasons_lib.api
```

If `reasons.api` ever imports one of those modules at module load time, a circular import could occur.

---

## Flow

1. A consumer imports `reasons_lib.api`.
2. Python loads `reasons.api` if it is not already in `sys.modules`.
3. The star import copies the public export set of `reasons.api` into `reasons_lib.api`’s globals.
4. The `reasons_lib.api` module object is cached in `sys.modules`.
5. Later attribute access such as `reasons_lib.api.some_name`, or `from reasons_lib.api import some_name`, resolves to the object bound during the star import.

There is no runtime call, no validation step, and no forwarding logic.

---

## Invariants

- `reasons.api` must be importable. If it cannot be imported, importing `reasons_lib.api` fails.
- Any name intended to remain available through the legacy path must be part of `reasons.api`’s star-import export set:
  - included in `reasons.api.__all__` if `__all__` is defined; or
  - a non-underscore global in `reasons.api` otherwise.
- The shim does not own API behavior. Re-exported functions, classes, and constants are the same objects as those in `reasons.api`, but rebinding a name on `reasons_lib.api` will not change `reasons.api`.
- The shim’s re-exported surface is a snapshot at import time. Names added to `reasons.api` later are not automatically visible through an already-imported `reasons_lib.api`.
- The shim does not define its own `__all__`. Therefore `from reasons_lib.api import *` uses Python’s default rule for the shim: export all non-underscore names currently present in `reasons_lib.api`.
- The import graph should avoid cycles: `reasons_lib.api -> reasons.api` must not loop back through a module that imports `reasons_lib.api`.

---

## Error Handling

- There is no explicit error handling.
- If `reasons.api` raises during import, the exception propagates unchanged.
- If `reasons.api` is missing, `ImportError` propagates.
- If a consumer requests a name that was not re-exported, Python raises:
  - `ImportError` for `from reasons_lib.api import missing_name`
  - `AttributeError` for `reasons_lib.api.missing_name`
- No deprecation warning is emitted. The shim is silent.
- No fallback implementation exists. This file does not try to import from an older `reasons_lib` implementation if `reasons.api` fails.
- Lint errors are suppressed only for the intentional unused/star import.

---

## Topics to Explore

- [file] `reasons/api.py` — Defines the canonical API surface and, if present, the `__all__` list that controls what this shim re-exports.
- [file] `reasons/ask.py` — One canonical module listed as importing the legacy shim; inspect why canonical code still depends on `reasons_lib.api` and whether that creates cycle risk.
- [file] `tests/test_api.py` — Shows the expected compatibility behavior and what API names consumers rely on.
- [file] `reasons_lib/__init__.py` — Package-level compatibility behavior; the package initializer may also re-export or alias legacy names.
- [general] `legacy-namespace-migration` — Whether `reasons_lib` is formally deprecated, when it can be removed, and whether a warning should be emitted.

## Beliefs

- `reasons-lib-api-is-compat-shim` — `reasons_lib/api.py` contains no API implementation and only re-exports `reasons.api` via a star import.
- `reasons-lib-api-requires-reasons-api-importability` — Importing `reasons_lib.api` fails if `reasons.api` cannot be imported; there is no fallback implementation.
- `reasons-lib-api-exports-reasons-api-public-names` — The names available on `reasons_lib.api` are determined by `reasons.api`’s star-import export set at import time.
- `reasons-lib-api-has-no-local-all-or-getattr` — The shim does not define `__all__`, `__getattr__`, or deprecation warnings.
- `reasons-lib-api-is-imported-by-canonical-modules` — Canonical modules such as `reasons/ask.py` and `reasons/cli.py` import this legacy shim, so it is part of internal runtime imports, not just external compatibility.
