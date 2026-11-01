---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/repair.py

# `reasons_lib/repair.py`

## Purpose

This file is a **backwards-compatibility shim** — a single-line re-export that exists solely so that code still importing from the old package path (`reasons_lib`) continues to work after the canonical package was renamed to `reasons`. Its sole responsibility is to make `from reasons_lib.repair import X` resolve to the same objects as `from reasons.repair import X`.

The docstring is the authoritative statement of intent:

```python
"""Backwards-compatibility shim — use ``from reasons.repair`` instead."""
```

There is no logic, no state, no configuration here. The file is a transitional artifact of the `reasons_lib` → `reasons` package rename.

## Key Components

There are no classes, functions, or constants defined in this file. The entire implementation is:

```python
from reasons.repair import *  # noqa: F401,F403
```

- **`from reasons.repair import *`** — Pulls every public name (anything without a leading underscore, or anything listed in `__all__`) from the canonical module into this module's namespace.
- **`# noqa: F401,F403`** — Suppresses two flake8/pyflakes warnings:
  - `F401` — "module imported but unused" (the star import looks unused to the linter)
  - `F403` — "star-style import makes it unclear which names are exported"

These noqa comments are **load-bearing** — without them, the linter would flag this file as a code smell. The suppression is intentional and correct for a shim.

## Patterns

| Pattern | Detail |
|---|---|
| **Delegation / façade** | This module delegates 100 % of its namespace to `reasons.repair`. It adds no behaviour. |
| **Star-re-export shim** | A common idiom during package renames: keep the old dotted path alive with `import *` so downstream `from old_pkg.mod import X` statements don't break. |
| **`noqa` on star imports** | The standard way to silence linters when a star import *is* the entire point of the file. |
| **Parallel package tree** | The repo contains both `reasons/` and `reasons_lib/` with near-identical file lists. Every file in `reasons_lib/` that also exists in `reasons/` is expected to be a shim of this same shape. The one exception is `reasons/merkle.py`, which has no `reasons_lib/` counterpart — suggesting `merkle` was added *after* the rename. |

## Dependencies

**Imports (outbound):**
- `reasons.repair` — the single, sole dependency. All public symbols available in `reasons_lib.repair` are exactly those exposed by `reasons.repair`.

**Imported by (inbound):**
- `reasons/api.py` — the main API surface; imports from `reasons_lib.repair` (likely a legacy import path that hasn't been migrated yet)
- `reasons_lib/repair_premises.py` — the sibling shim for premise repair; its star-import chain may pull names through this module
- `tests/test_repair.py`, `tests/test_repair_premises.py`, `tests/test_research.py` — test suites that import repair-related helpers; some may still use the old `reasons_lib` path

> **Maintenance note:** Once every importer has been switched to `from reasons.repair import …`, this file (and its siblings in `reasons_lib/`) can be deleted. The `noqa` comments become irrelevant at that point.

## Flow

There is no runtime logic. Execution is:

1. Python's import machinery encounters `from reasons_lib.repair import *`.
2. It executes `reasons_lib/repair.py`, which in turn triggers `from reasons.repair import *`.
3. The canonical `reasons/repair.py` module is loaded (if not already in `sys.modules`), and all its public names are copied into the `reasons_lib.repair` namespace.
4. The original importer's star-import or explicit-name lookup resolves against those copied names.

No functions are called, no objects are constructed, no I/O occurs.

## Invariants

- **Namespace equivalence:** `dir(reasons_lib.repair)` (excluding `__` dunders) must be a superset of `dir(reasons.repair)`. In practice they are identical because `import *` copies the full public set.
- **No divergence:** This file must never define its own names. If a developer adds a function here, they are breaking the shim contract and should instead edit `reasons/repair.py`.
- **One-directional dependency:** `reasons_lib` depends on `reasons`, never the reverse. This prevents circular imports.

## Error Handling

There is no error handling in this file. Errors propagate transparently:

- If `reasons.repair` fails to import (syntax error, missing dependency, circular import), the exception surfaces at the original import site and the traceback will show both `reasons_lib/repair.py` and `reasons/repair.py` frames.
- If a downstream consumer does `from reasons_lib.repair import some_function` and `some_function` was removed from `reasons.repair`, they get a `NameError` (or `ImportError` for star-import resolution) — identical to what they'd get from the canonical path.
- There are no `try/except` blocks, no fallbacks, no logging. The shim is either fully functional or the import fails.

## Topics to Explore

- [file] `reasons/repair.py` — The canonical implementation this shim delegates to; understanding its API surface tells you exactly what `reasons_lib.repair` exposes.
- [file] `reasons/repair_premises.py` — Sibling repair logic for premises; its relationship to `repair.py` clarifies the module boundaries in the repair subsystem.
- [repo] `reasons_lib/` vs `reasons/` — The full parallel tree makes the migration strategy visible; comparing which files exist in one but not the other (e.g., `merkle.py`) dates when the rename happened.
- [general] `package-rename-migration` — How the project manages the two-package coexistence: whether `pyproject.toml` registers both, whether CI lints for `reasons_lib` imports, and what the removal criteria are.
- [file] `reasons/api.py` — The primary public API that imports from this shim; tracing its imports shows the remaining `reasons_lib` references that block shim removal.

## Beliefs

- `reasons-lib-repair-is-a-reason-repair` — `reasons_lib/repair.py` is a namespace alias for `reasons/repair.py`; they expose an identical set of public names and one must never define symbols the other does not.
- `reasons-lib-repair-replaces-direct-import` — Any code path that currently does `from reasons_lib.repair import X` will resolve identically if rewritten to `from reasons.repair import X`, with no behavioural difference.
- `reasons-lib-has-no-unique-modules` — Every module present in `reasons_lib/` has a corresponding file in `reasons/`; the `reasons_lib` tree adds no functionality of its own (the sole exception in the repo tree is `reasons/merkle.py`, which postdates the rename and has no `reasons_lib` counterpart).
- `shim-removal-is-safe-when-no-importers-remain` — Deleting `reasons_lib/repair.py` is safe if and only if none of the five known importers (`reasons/api.py`, `reasons_lib/repair_premises.py`, `tests/test_repair.py`, `tests/test_repair_premises.py`, `tests/test_research.py`) reference the `reasons_lib.repair` path.
