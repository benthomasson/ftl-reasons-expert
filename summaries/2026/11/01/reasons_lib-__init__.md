---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/__init__.py

# `reasons_lib/__init__.py`

## 1. Purpose

This file is a **backwards-compatibility shim**. The project was originally developed under the package name `reasons_lib` and was later renamed to `reasons`. Rather than breaking every downstream consumer that still does `import reasons_lib` or `from reasons_lib import X`, this `__init__.py` re-exports the entire public API of the canonical `reasons` package into the `reasons_lib` namespace.

Its single responsibility: **make `reasons_lib` a transparent alias for `reasons`**.

## 2. Key Components

There is exactly one statement in this file:

```python
from reasons import *  # noqa: F401,F403
```

| Element | Role |
|---|---|
| `from reasons import *` | Re-exports every public name (those not starting with `_` and listed in `__all__` if defined) from the `reasons` package's `__init__.py` into `reasons_lib`'s namespace. |
| `# noqa: F401` | Suppresses the flake8 "imported but unused" warning — the import *is* used, it's intentionally re-exported. |
| `# noqa: F403` | Suppresses the flake8 "star import" warning — the star import is deliberate for a compatibility shim. |

The docstring on line 1 makes the intent explicit to any developer who opens the file:

```python
"""Backwards-compatibility shim — use ``import reasons`` instead."""
```

## 3. Patterns

- **Compatibility shim / alias package.** A well-known Python idiom for surviving a package rename. The old name (`reasons_lib`) becomes a thin re-export layer over the new name (`reasons`), so existing `pip install` users, `import` statements, and IDE autocompletions keep working with zero changes on their side.
- **`__init__.py` as the public surface.** Both `reasons/__init__.py` and `reasons_lib/__init__.py` act as the "front door." The shim delegates *all* of that responsibility upstream rather than cherry-picking symbols, which means the two namespaces stay in lockstep automatically.
- **Mirror directory structure.** Notice that `reasons_lib/` contains near-identical submodules (`api.py`, `ask.py`, `cli.py`, etc.) alongside `reasons/`. The `__init__.py` shim handles the top-level re-export; the per-module mirrors (e.g. `reasons_lib/api.py`) likely do the same trick at the submodule level (e.g. `from reasons.api import *`), so `from reasons_lib.api import X` also works.

## 4. Dependencies

**Upstream (this file imports):**
- `reasons` — the canonical package. This is a hard dependency; if `reasons` is not importable, `reasons_lib` will fail at import time with `ModuleNotFoundError`.

**Downstream (what imports this file):**
- Any legacy consumer that does `import reasons_lib`, `from reasons_lib import SomeClass`, or `from reasons_lib import *`.
- Test files that reference the old name (e.g., anything in `tests/` that uses `reasons_lib.*`).
- External packages or user code that were written against the pre-rename API.

**Sibling modules in `reasons_lib/`:** Each submodule (`reasons_lib/api.py`, `reasons_lib/cli.py`, etc.) is almost certainly a parallel shim that re-exports from the corresponding `reasons.*` module. The `__init__.py` alone does *not* make `reasons_lib.api` importable — that requires `reasons_lib/api.py` to exist and do its own `from reasons.api import *`.

## 5. Flow

Execution is trivially a single step:

1. Python encounters `import reasons_lib` (or `from reasons_lib import X`).
2. The import machinery executes `reasons_lib/__init__.py`.
3. The `from reasons import *` statement triggers Python to first fully execute `reasons/__init__.py` (which may itself import submodules like `reasons.api`, `reasons.storage`, etc.).
4. Every public name from `reasons`'s namespace is copied into `reasons_lib`'s namespace.
5. Subsequent attribute access (`reasons_lib.SomeClass`) resolves against that copied namespace.

There is no conditional logic, no lazy loading, no data transformation — it's a static namespace copy at import time.

## 6. Invariants

- **Namespace parity:** After a successful import, `reasons_lib.<name>` and `reasons.<name>` refer to the *same* object (not a copy). Mutating a module-level constant via one alias is visible through the other.
- **`reasons` must be installed and importable** before `reasons_lib` can be imported. There is no fallback or stub.
- **The shim must never define its own symbols.** Adding a class, function, or constant directly in `reasons_lib/__init__.py` would shadow the corresponding `reasons` name and break parity. The `# noqa` comments signal that the *only* intended content is the star re-export.

## 7. Error Handling

There is **no explicit error handling** in this file. Errors surface as:

- **`ModuleNotFoundError: No module named 'reasons'`** — if the canonical package is missing from the environment (e.g., a partially installed wheel). This propagates up to the caller's `import reasons_lib` statement.
- **`ImportError`** — if `reasons/__init__.py` itself has an import failure in one of its submodules. The error message will reference the *original* failing module inside `reasons`, not `reasons_lib`, which can make debugging slightly confusing for users who only see the shim in their stack trace.

The shim swallows nothing; it defers all error semantics to `reasons`.

---

## Topics to Explore

- [file] `reasons/__init__.py` — The canonical package's `__init__.py` determines exactly *which* names the `*` import re-exports; understanding its `__all__` (if defined) or its top-level definitions is essential to knowing what `reasons_lib` actually surfaces.
- [file] `reasons_lib/api.py` — A representative submodule shim; reading it confirms whether each subpackage-level mirror follows the same `from reasons.api import *` pattern or uses a more explicit re-export list.
- [repo] `ftl-reasons/CHANGELOG.md` — The changelog will document the rename from `reasons_lib` to `reasons`, the version at which it happened, and any deprecation timeline for the shim.
- [file] `pyproject.toml` — Defines the package name, entry points, and whether `reasons_lib` is registered as an additional top-level package (via `[tool.setuptools.packages.find]` or `packages` list), which is critical for understanding how both namespaces are installed.
- [general] `deprecation-timeline` — Whether and when `reasons_lib/` will be removed entirely; knowing this helps you decide whether to write new code against `reasons` or still support `reasons_lib` consumers.

## Beliefs

- `reasons-lib-re-exports-reasons` — `reasons_lib/__init__.py` contains no logic of its own; every public name it exposes is a direct reference to the same object defined in the `reasons` package, so the two namespaces are structurally identical after import.
- `reasons-lib-requires-reasons-installable` — Importing `reasons_lib` will raise `ModuleNotFoundError` if the `reasons` package is not present in the current Python path; there is no fallback, stub, or lazy-import mechanism.
- `reasons-lib-is-a-reasons-substitute` — The shim exists solely to preserve the pre-rename import surface; any new feature added to `reasons` is automatically visible through `reasons_lib` without touching the shim, but removing a name from `reasons` silently breaks `reasons_lib` consumers.
- `reasons-lib-star-import-skips-underscored-names` — The `from reasons import *` construct excludes any name beginning with `_` (and names not in `__all__` if one is defined), so private helpers in `reasons` are *not* re-exported through `reasons_lib`.
- `reasons-lib-contains-mirror-submodules` — The `reasons_lib/` directory holds per-module shim files (`api.py`, `cli.py`, `storage.py`, etc.) in addition to `__init__.py`, meaning `from reasons_lib.api import X` requires `reasons_lib/api.py` to exist and perform its own re-export; the `__init__.py` alone does not make submodules accessible.
