---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/pg.py

# `reasons_lib/pg.py`

## Purpose

This file owns **no logic at all**. It is a one-line backwards-compatibility shim whose only job is to keep the import path `reasons_lib.pg` working while the real module lives at `reasons.pg`.

The docstring states the intent unambiguously:

```python
"""Backwards-compatibility shim — use ``from reasons.pg`` instead."""
from reasons.pg import *  # noqa: F401,F403
```

Concretely: any code (or plugin) that historically did `from reasons_lib import pg` or `import reasons_lib.pg` will continue to resolve, receiving the exact same objects exposed by `reasons.pg`. It is part of the broader `reasons_lib/` façade (a whole second tree of matching filenames that mirror `reasons/`).

## Key Components

This file has exactly two meaningful fragments:

| Element | Contract |
|---|---|
| Module docstring | Explicitly tells developers the canonical location is `reasons.pg`. Not enforced by Python — purely documentation. |
| `from reasons.pg import *` | Re-exports every public name from `reasons.pg` into `reasons_lib.pg`'s namespace. |
| `# noqa: F401,F403` | Suppresses flake8 "imported but unused" (`F401`) and "star import" (`F403`) — signals that the star import is intentional, not accidental. |

There are **no** classes, functions, constants, or `__all__` defined in this module.

## Patterns

- **Compatibility shim / façade**: The classic pattern of leaving a thin module at the old import path pointing at the new one. Central to any package rename or split (e.g., `package` → `package_lib` with `package` adopting the canonical name).
- **Star re-export**: `from x import *` is the shortest way to make the shim "transparent" — callers see the same symbols without having to add explicit import-forwarding lines for every name.
- **`# noqa` as a design assertion**: By suppressing F401/F403, the maintainer documents that a lint violation here is expected and desired.
- **Parallel-tree façade**: The existence of the entire second tree (`reasons_lib/{api,ask,cli,...}.py` matching `reasons/{...}.py`) strongly implies a mechanical policy: every module in `reasons/` has a mirror shim in `reasons_lib/`, except a few comonly not mirrored.

## Dependencies

**Imports**
- `reasons.pg` — the real implementation. If this module fails to import, `reasons_lib/pg.py` fails at *import* time (not first-use time).
- Indirectly, whichever modules `reasons/pg.py` itself imports (e.g., `reasons/api.py` or `reasons/storage.py` may be involved — see `Imports`/`Imported By` in the repo graph).

**Imported By** (per the context graph)
- `reasons/api.py`
- `reasons/pg.py` (⚠ this listed edge is almost certainly a static-analysis artifact: `reasons.pg` and `reasons_lib.pg` share the same leaf name, so name-based tooling reports them as mutually importing. Treat `reasons_lib/pg.py` as *not* genuinely imported by `reasons/pg.py` — otherwise you'd have an import cycle at load time).
- `tests/conftest.py`
- `tests/test_pg.py`

The practical contract: removing this file breaks any downstream that imports `reasons_lib.pg`, but nothing in the `reasons/` package should depend on it.

## Flow

Runtime flow is trivial:

1. Interpreter imports `reasons_lib.pg`.
2. Executes `from reasons.pg import *`.
   - Python imports `reasons.pg` (which may in turn execute its own `__init__` imports, submodule loads, etc.).
   - The star-import then binds into `reasons_lib.pg`'s globals:
     - If `reasons.pg` defines `__all__`, only names listed there are exported.
     - Otherwise, every name not starting with `_` (and not a module-level dunder) is copied.
3. The module finishes; callers see the same function/class *objects* (not copies) as `reasons.pg`.

There is no state, no config, no data transformation — the shim is semantically equivalent to `reasons_lib.pg is not reasons.pg` but every attribute on the former points to the same object as the latter.

## Invariants

Enforced or relied upon:

1. **`reasons.pg` must be importable.** If a refactor moves/deletes the real module, the shim will raise `ModuleNotFoundError` on import.
2. **`reasons.pg.__all__` (or its public surface) must remain stable.** Because `import *` is used, any name that becomes private (leading underscore) or is omitted from `__all__` in `reasons.pg` will silently disappear from `reasons_lib.pg`. This is the shim's main hidden dependency.
3. **No names are re-declared locally.** Because the shim does not set `__all__`, its own `__all__` is empty/undefined — so `from reasons_lib.pg import *` re-exports whatever the star-import pulled in without further filtering. This is usually fine but worth knowing.
4. **Do not add logic here.** Any behavior added to the shim would not exist in the canonical `reasons.pg` and would silently diverge two supposedly identical modules.

## Error Handling

- **Import errors**: pass through unmodified. A `ModuleNotFoundError` or `ImportError` from `reasons.pg` (or its own transitive imports) will surface at `import reasons_lib.pg` time with the original traceback.
- **Attribute errors**: not applicable; the module defines no attributes itself.
- **Lint errors**: suppressed by the `# noqa: F401,F403` comment — a deliberate exception, not swallowed logic.
- **No try/except**: the shim does not catch or wrap anything. Failures are propagated verbatim, which is correct for a shim (you never want a compatibility layer to hide a broken canonical module).

---

## Topics to Explore

- [file] `reasons/pg.py` — The actual implementation this shim re-exports; read its `__all__` (or public surface) to know exactly what `from reasons_lib.pg import *` exposes.
- [file] `tests/test_pg.py` — Exercises the `reasons.pg` behavior; useful as a spec for what should flow through the shim.
- [file] `tests/conftest.py` — Shows whether tests exercise `reasons_lib.pg` explicitly (validating the shim) or only `reasons.pg`.
- [general] `reasons-lib-package-shim-pattern` — Confirm whether every `reasons_lib/*.py` is a shim (as the tree strongly suggests) and identify the exceptions (e.g., `reasons_lib/merkle.py` missing while other modules match).
- [file] `reasons/api.py` — One of the listed importers of `pg`; good for seeing how a production module consumes the API that this shim forwards.

## Beliefs

- `reasons-lib-pg-is-a-re-export-shim` — `reasons_lib/pg.py` contains no logic beyond a docstring and `from reasons.pg import *`.
- `reasons-lib-pg-requires-reasons-pg` — Importing `reasons_lib.pg` fails at import time if `reasons.pg` is unimportable.
- `reasons-lib-pg-honors-reasons-pg-all` — Because it uses `import *`, the set of names exposed depends on `reasons.pg.__all__` if defined.
- `reasons-lib-pg-suppresses-lint-warnings` — The `# noqa: F401,F403` comment intentionally disables flake8 unused-import and star-import warnings.
- `reasons-lib-pg-is-a-backwards-compatibility-alias` — The docstring directs new code to `reasons.pg`, meaning `reasons_lib.pg` exists only to preserve older import paths.
