---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/api.py

# `reasons_lib/api.py`

## Purpose

This file is a **one-line backwards-compatibility shim**. It exists solely to preserve the old import path (`from reasons_lib.api import ...`) after the project was renamed from the `reasons_lib` package to the `reasons` package. Its docstring states the intent explicitly:

```python
"""Backwards-compatibility shim — use ``from reasons.api`` instead."""
```

It carries **no logic of its own**. Its single responsibility is to make `reasons_lib.api` a transparent alias for `reasons.api`, so that any downstream code (third-party consumers, older test files, internal modules that haven't been migrated) can continue to import symbols from the old location without breaking.

## Key Components

There is exactly one statement in the file:

```python
from reasons.api import *  # noqa: F401,F403
```

| Element | Role |
|---|---|
| `from reasons.api import *` | Re-exports every public name from the real `reasons.api` module into the `reasons_lib.api` namespace. |
| `# noqa: F401` | Suppresses the linter's "imported but unused" warning — the import *is* used, it's just used as a re-export. |
| `# noqa: F403` | Suppresses the linter's "star import" warning — intentional here; the shim must expose the full surface of `reasons.api` by construction. |

There are no functions, classes, constants, or type annotations. The file's entire contract is: **`dir(reasons_lib.api)` is a superset of `dir(reasons.api)` at the moment of import**.

## Patterns

- **Facade / Delegation Pattern (trivial form):** The module delegates 100 % of its namespace to another module. A consumer calling `reasons_lib.api.something()` is, after the star-import, invoking the same object that lives in `reasons.api.something()`. There is no wrapping, no monkey-patching, no conditional logic.
- **Package-rename migration shim:** The repo contains *two* top-level packages (`reasons/` and `reasons_lib/`) that mirror each other. `reasons_lib/` is the legacy name; `reasons/` is the current name. The shim is the standard "leave a re-export stub in the old path" technique so you can deprecate incrementally rather than breaking every import in a single commit.
- **Star-import as a re-export mechanism:** In Python, `from X import *` inside a module makes all of `X`'s `__all__` (or all public names) available as if they were defined in the importing module. This is the idiomatic way to create a "namespace alias" without writing individual `from reasons.api import foo` lines.

## Dependencies

### Upstream (what this file imports)

- **`reasons.api`** — the single and only dependency. At import time Python resolves `reasons.api`, executes that module, and then copies every public name into the `reasons_lib.api` namespace. If `reasons.api` raises on import, this shim fails identically.

### Downstream (what imports this file)

The "Imported By" list is long (50+ entries) and splits into two groups:

1. **Internal modules** (`reasons/ask.py`, `reasons/cli.py`, `reasons/contradictions.py`, `reasons/derive.py`, `reasons/mcp_server.py`, `reasons/pg.py`, `reasons/repair.py`) — these are modules that still reference the old `reasons_lib.api` path. They are candidates for migration to `from reasons.api import ...`.
2. **Tests** (the majority of the list) — most test files under `tests/` import through the shim, meaning the test suite itself is still wired to the legacy path.

> **Practical note:** Because the shim is a pure re-export, you can safely change every `from reasons_lib.api import X` to `from reasons.api import X` without any behavioural difference. The shim exists only to let that migration happen gradually.

## Flow

Execution is trivial and happens once at import time:

```
import reasons_lib.api
        │
        ▼
Python executes:  from reasons.api import *
        │
        ├── 1. Import & execute reasons/api.py  (the real module)
        ├── 2. Collect all public names (or __all__) from reasons.api
        └── 3. Bind each name into the reasons_lib.api module's __dict__
        │
        ▼
reasons_lib.api now mirrors reasons.api
```

After that, any `reasons_lib.api.some_function(...)` call is **the same Python function object** as `reasons.api.some_function`. There is no indirection, no proxy, no wrapper.

## Invariants

- **Namespace parity:** At any point after a successful import, every name accessible via `reasons_lib.api` is also accessible (and identical) via `reasons.api`. The shim adds nothing; it only reflects.
- **No side effects beyond import:** The shim does not call any function, read config, or modify globals. Its only effect is the star-import binding.
- **Ordering:** `reasons.api` must be importable before `reasons_lib.api` can be imported (obviously, since it's a dependency). There is no circular-import risk *introduced by this file*, but if `reasons.api` itself imports from `reasons_lib.something`, you'd have a cycle — and the "Imported By" list does show `reasons/` modules importing `reasons_lib.api`, so this is a live consideration.
- **`__all__` governs the surface:** If `reasons.api` defines `__all__`, only those names are re-exported. If it doesn't, every name not starting with `_` is re-exported. Changing `__all__` in `reasons.api` silently changes what `reasons_lib.api` exposes.

## Error Handling

- **Import-time failure:** If `reasons.api` raises (missing dependency, syntax error, circular import), the star-import propagates that exception up through `reasons_lib.api`. The consumer sees the traceback pointing at `reasons/api.py`, which is correct — the shim has nothing to hide.
- **Attribute-missing at call time:** If a consumer does `reasons_lib.api.nonexistent_fn()`, they get a standard `AttributeError`. There is no custom error message, no deprecation warning, no `__getattr__` hook. The shim does not intercept attribute access.
- **No exceptions are generated by this file itself.** It cannot raise beyond what `reasons.api` raises during import.

> **Tip for maintainers:** If you want to add a deprecation warning (e.g., "use `reasons.api` instead"), you would need to replace the star-import with an explicit `__getattr__` on the module that logs a warning before falling back to the real attribute. As written, the shim is silent.

## Topics to Explore

- [file] `reasons/api.py` — The real API surface this shim re-exports; understanding its public names, `__all__`, and function signatures is the actual work.
- [file] `reasons_lib/__init__.py` — Check whether the package-level `__init__` also acts as a shim or does additional re-exports, and whether it sets `__all__` at the package level.
- [general] `package-rename-migration` — The broader `reasons_lib → reasons` rename: which files are fully migrated, which still use the old path, and what the removal plan looks like (likely a future PR that deletes `reasons_lib/` entirely).
- [file] `tests/test_api.py` — The primary test for the API surface; shows what the public contract actually looks like and whether it tests through `reasons.api` or `reasons_lib.api`.
- [diff] `reasons_lib/ vs reasons/` — Compare the two package directories to confirm they are true mirrors (note: `reasons/` has `merkle.py` and `storage.py` that `reasons_lib/` lacks, suggesting `reasons_lib/` is already stale).

## Beliefs

- `reasons_lib-api-replaces-reasons-api` — `reasons_lib/api.py` is a pure re-export of `reasons.api`; deleting it and updating imports produces zero behavioural change.
- `reasons-lib-is-stale` — The `reasons_lib/` package is missing modules that exist in `reasons/` (`merkle.py`, `storage.py`), indicating the legacy package is not kept in sync and is slated for removal.
- `star-import-scope-depends-on-all` — The set of names exposed by `reasons_lib.api` is determined entirely by `reasons.api`'s `__all__` (or its public namespace); adding a function to `reasons.api` without updating `__all__` will silently change what the shim exposes.
- `shim-introduces-no-encapsulation` — Because the star-import binds the same function objects, any monkey-patch applied to `reasons.api` (or `reasons_lib.api`) is visible through the other; there is no isolation between the two namespaces.
- `importers-still-use-legacy-path` — 7 internal modules under `reasons/` and 50+ test files still import via `reasons_lib.api`, meaning the shim is load-bearing until every one of those imports is migrated to `reasons.api`.
