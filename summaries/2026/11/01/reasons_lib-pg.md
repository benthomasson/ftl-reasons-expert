---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/pg.py

# `reasons_lib/pg.py`

## Purpose

This file is a **backwards-compatibility shim**. Its sole job is to let legacy import paths (`from reasons_lib.pg import ...`) continue to resolve after the project was restructured from a `reasons_lib/` package into a `reasons/` package. It contains no original logic — it is a one-line re-export of everything defined in `reasons/pg.py`.

In practical terms: if any downstream code, test fixture, or third-party integration still writes `from reasons_lib.pg import PGStore` (or whatever symbols live in `reasons/pg.py`), Python resolves the name through this shim without any modification to that caller.

## Key Components

There are **no original classes, functions, or constants** in this file. The entire module is:

```python
from reasons.pg import *  # noqa: F401,F403
```

- **`from reasons.pg import *`** — pulls every public name (those without a leading underscore, or those listed in `__all__`) from the real implementation module into the `reasons_lib.pg` namespace.
- **`# noqa: F401,F403`** — silences `pyflakes`/`ruff` warnings:
  - `F401` — "module imported but unused" (the `*` binding is never referenced locally)
  - `F403` — "star import makes it impossible to tell what names are defined"

The docstring (`"""Backwards-compatibility shim — use ``from reasons.pg`` instead."""`) is the only "documentation" and it explicitly tells future readers to migrate their imports.

## Patterns

| Pattern | How it appears here |
|---|---|
| **Shim / facade** | A thin proxy module that delegates entirely to the canonical module. No state, no logic. |
| **Package rename migration** | The repo mirrors every file from `reasons/` into `reasons_lib/` as a one-liner re-export, preserving old import paths during a transition. |
| **`noqa` suppression for intentional star import** | The linter is *correct* to flag a bare `import *`; the `noqa` tells the reader "this is deliberate, not sloppiness." |

This is a well-known idiom in Python projects that rename their top-level package (e.g., `pip` → `pip-tools`, `django` → `django-rest-framework`). The shim files are expected to be **deleted** once all internal call-sites and known external consumers have migrated.

## Dependencies

### Upstream (this file imports)
- **`reasons.pg`** — the real implementation. All symbols (classes, functions, constants) that `reasons.pg` exposes publicly are re-exported. This is the *only* dependency.

### Downstream (imported by)
- `reasons/api.py` — likely a test or internal module that still uses the old path.
- `reasons/pg.py` — this entry is unusual; it may be a circular reference in the import graph or a test-utility that imports the shim to verify it works.
- `tests/conftest.py` — test fixtures that set up PostgreSQL-backed storage may still `from reasons_lib.pg import ...`.
- `tests/test_pg.py` — the PG-specific test suite.

> **Maintenance note:** The fact that `reasons/api.py` and `reasons/pg.py` appear as importers of the *shim* (rather than the shim importing them) suggests those files have not yet been migrated to the new `reasons.pg` path. Once they are, the shim's sole purpose vanishes.

## Flow

There is no execution logic to trace. At **import time** the Python interpreter:

1. Loads `reasons_lib/pg.py`.
2. Encounters `from reasons.pg import *`.
3. Imports `reasons.pg` (executing that module once, caching it in `sys.modules`).
4. Iterates over `reasons.pg.__dict__`, copies every name that doesn't start with `_` (or every name in `__all__` if defined) into the local `reasons_lib.pg` namespace.
5. Any subsequent `from reasons_lib.pg import X` resolves `X` directly from that namespace — **no further indirection**.

At **runtime**, calls to any imported symbol execute inside `reasons.pg` as if the shim didn't exist. There is zero runtime overhead beyond the one-time `*` unpacking at import.

## Invariants

- **Transparency:** Every public name in `reasons.pg` must be accessible as a name in `reasons_lib.pg`. If `reasons.pg` adds a new public symbol, the shim picks it up automatically (because `import *` is evaluated at import time). If `reasons.pg` *removes* a symbol, the shim silently loses it.
- **No shadowing:** This file defines no names of its own, so it cannot shadow or override anything from `reasons.pg`.
- **Single source of truth:** `reasons/pg.py` is canonical. This file must never contain logic; doing so would create a divergence risk.

## Error Handling

There is **none** in this file. All error semantics (connection failures, schema errors, query exceptions) are defined and raised inside `reasons/pg.py` and propagate unchanged through the shim. If `reasons.pg` fails to import (e.g., missing `psycopg2`), the `from reasons.pg import *` line raises `ImportError` / `ModuleNotFoundError` at import time, and the shim never finishes loading.

---

## Topics to Explore

- [file] `reasons/pg.py` — The actual PostgreSQL storage implementation that this shim re-exports; understanding its public API tells you exactly what surface area the shim must preserve.
- [file] `reasons/storage.py` — The likely abstraction layer that `pg.py` plugs into; comparing the two clarifies how the PG backend is selected vs. other storage backends.
- [general] `package-rename-migration` — How the project transitioned from `reasons_lib/` to `reasons/`; searching for remaining `reasons_lib` imports across the repo will show how much migration work is left before the shims can be deleted.
- [file] `tests/test_pg.py` — The test suite that exercises this module; reveals what the public contract of the PG layer looks like and which symbols are actually consumed through the shim.
- [file] `pyproject.toml` — Package metadata, dependencies (e.g., `psycopg`/`asyncpg`), and whether both `reasons` and `reasons_lib` are declared as installable packages.

---

## Beliefs

- `reasons_lib/pg.py-replaces-reasons/pg.py-import-path` — The shim exists solely so that `from reasons_lib.pg import X` is semantically equivalent to `from reasons.pg import X`; it adds no behavior of its own.
- `reasons_lib/pg.py-requires-reasons/pg.py` — If `reasons.pg` fails to import, this module raises at import time; the shim cannot function independently.
- `reasons_lib/pg.py-picks-up-public-names-automatically` — Because it uses `import *`, any new public symbol added to `reasons.pg` is immediately available through the shim without editing this file; conversely, removing a public symbol from `reasons.pg` silently removes it here.
- `reasons/api.py-still-imports-via-shim` — At least one module inside the *new* `reasons/` package (`api.py`) still routes through the old `reasons_lib` path, meaning the shim cannot yet be deleted.
- `shim-namespace-has-no-own-attributes` — The module `__dict__` of `reasons_lib.pg` contains only the re-exported names (plus builtins like `__name__`, `__doc__`); no function, class, or constant is defined locally.
