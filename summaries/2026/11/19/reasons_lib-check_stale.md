---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/check_stale.py

# `reasons_lib/check_stale.py` — Explanation

## 1. Purpose

This file is a **backwards-compatibility shim**. The project has been reorganized so that `check_stale` lives in the canonical package `reasons`, but an older package path `reasons_lib` still exists in the tree to keep legacy imports working.

Concretely, the file's only job is to make:

```python
from reasons_lib.check_stale import <anything>
```

behave exactly like:

```python
from reasons.check_stale import <anything>
```

The docstring is explicit about the direction of the migration:

> `"""Backwards-compatibility shim — use ``from reasons.check_stale`` instead."""`

This file owns **no behavior** — no functions, no classes, no constants, no side effects beyond loading the canonical module. Its responsibility is purely import-name resolution.

## 2. Key Components

There are no functions, classes, or constants in this file. The entire module body is:

- A **module docstring** (`"""Backwards-compatibility shim …"""`) — documentation and a hint to `help()` / IDEs that this path is deprecated.
- A **single star import** (`from reasons.check_stale import *`) — rebinds every name exported by the canonical module into this module's namespace.
- A **`# noqa: F401,F403` comment** — silences two pyflakes/flake8 warnings on that line:
  - `F401`: *module imported but unused* — because the module body appears to do nothing with the import.
  - `F403`: *`from module import *` used; unable to detect undefined names* — because a star import makes static analysis of the module's namespace impossible.

The API contract of `reasons_lib.check_stale` is therefore **identical to** the public API of `reasons.check_stale` — public names are the same objects, not copies:

```python
import reasons.check_stale as canonical
import reasons_lib.check_stale as legacy

assert legacy.anything is canonical.anything   # identity holds
```

For the actual classes/functions/contract of the stale-check engine, see `reasons/check_stale.py` — this shim is a transparent forwarding layer only.

## 3. Patterns

- **Facade/re-export shim (module alias).** The classic Python idiom for renaming a module across a major version without breaking downstream code: a stub module that forwards its namespace to the real implementation.
- **Star-import re-export.** The shim chooses `from X import *` rather than explicitly enumerating names. This means future additions to `reasons.check_stale`'s public API are automatically forwarded here with no edit required.
- **`# noqa` as a linter contract.** The comment is a deliberate signal that the star import is the *intended* behavior, not an oversight, and that the two pyflakes codes are contractually waived.
- **Mirror package layout.** The full `reasons_lib/` subtree mirrors `reasons/` file-for-file — every `reasons/<x>.py` has a counterpart `reasons_lib/<x>.py` that is a shim. This file is one member of that coordinated pattern (see `reasons_lib/__init__.py` for the same idea at package level).

## 4. Dependencies

**Depends on (imports):**
- `reasons.check_stale` — the canonical implementation. This is the only import.

**Depended on by (`imported_by` per repo analysis):**
- `reasons/api.py`
- `reasons/pg.py`
- `reasons/verify.py`
- `tests/test_check_stale.py`
- `tests/test_check_stale_issue25.py`
- `tests/test_pin_sources.py`

Notably, *internal* `reasons/*` modules appear as consumers. That should be treated as legacy debt: the canonical code should import from `reasons.check_stale` directly. When the migration completes, this shim and its consumers should collapse. Until then, any removal of this file will break those imports.

**Notably absent:** no third-party dependencies, no `__all__`, no runtime package metadata. The shim has zero footprint beyond forwarding.

## 5. Flow

Control flow at import time is a straight line — there is no branching, no lazy indirection, and no callable logic:

1. Interpreter begins executing `reasons_lib/check_stale.py`.
2. Executes `from reasons.check_stale import *`:
   - Imports (or retrieves from `sys.modules`) `reasons.check_stale`.
   - Reads its `__all__` if defined; otherwise takes every name not starting with `_`.
   - Injects each resolved name into the current module's globals.
3. Module finishes executing. `import reasons_lib.check_stale` now returns the shim module whose public names reference the real objects in `reasons.check_stale`.

Key consequence: `import reasons_lib.check_stale` **eagerly triggers** the full import of `reasons.check_stale` and all of *its* dependencies. There is no deferred or lazy loading — a broken/mising canonical module surfaces immediately to legacy importers.

**`__all__` behavior** — since we can't see `reasons/check_stale.py`'s contents from this file alone, both cases are valid:
- If `reasons/check_stale.py` defines `__all__`, only those names are forwarded.
- If it does not, every non-underscored name is forwarded — including any incidental names captured by its own imports (e.g., `json`, `os`, `typing` if imported there). This "namespace pollution" is harmless for legacy callers but is one reason explicit `__all__` is preferable in the canonical module.

## 6. Invariants

- **Public API identity.** Every public symbol reachable as `reasons_lib.check_stale.X` must be the *same object* as `reasons.check_stale.X` — no wrappers, no copies, no re-implementation. Tests importing from the shim exercise the canonical code.
- **Direction of dependency is one-way.** The shim imports from `reasons.check_stale`, never the reverse. If `reasons/check_stale.py` were changed to `from reasons_lib.check_stale import …`, the result is a circular import and an `ImportError` (partially initialized module).
- **Canonical is a superset of legacy.** Because the shim re-exports the current namespace, `reasons.check_stale` must retain any public name legacy callers still import. Removing such a name silently breaks `reasons_lib.check_stale` consumers.
- **No new API is added.** The shim must not define functions/classes/constants; otherwise it would diverge from the canonical path and future migrations would need to inspect both.
- **Underscore names are not forwarded** (unless the canonical module puts them in `__all__`). Internal helpers like `_resolve_source()` are intentionally not part of the shim's surface.
- **Lint waiver is line-scoped and intentional.** `# noqa: F401,F403` must remain on the star-import line; removing it will turn CI lint failures into noise but not change runtime behavior.

## 7. Error Handling

There is **no error handling** in this file — no `try/except`, no custom exceptions, no defensive logic. Errors are simply *propagated*:

- **`ImportError` / `ModuleNotFoundError`** if `reasons.check_stale` is deleted/renamed — raised from the `import` statement during shim import, so it surfaces to the caller as `import reasons_lib.check_stale` failing.
- **`SyntaxError` or `AttributeError`** originating inside `reasons/check_stale.py` — propagate through the shim unchanged (the traceback will show both the shim frame and the canonical frame).
- **`ImportError` from circular imports** if the canonical module ever starts importing `reasons_lib.check_stale` (see invariant above).
- **No swallowing, no logging, no fallback.** The shim's philosophy is "the canonical module's import-time behavior *is* the shim's import-time behavior."

A subtle consequence: because the shim eagerly loads the canonical module, debugging failures through the legacy path can produce a traceback that ends in `reasons/check_stale.py` even though the user imported from `reasons_lib.check_stale`. This is expected and correct — it confirms the shim is doing its job.

---

## Topics to Explore

- [file] `reasons/check_stale.py` — The canonical implementation; this is where the actual stale-check logic, signatures, and constants live. Everything in this shim forwards here.
- [file] `reasons_lib/__init__.py` — Shows whether the mirror package uses the same star-import shim convention at the package level, which affects `import reasons_lib` semantics and `__all__` propagation.
- [file] `tests/test_check_stale_issue25.py` — Indicates a specific regression scenario the stale logic is expected to handle; useful for understanding edge cases in the canonical module's contract.
- [function] `reasons/check_stale.py:__all__` — Check whether the canonical module defines `__all__`; this determines exactly which names this shim re-exports and whether incidental imports leak through.
- [general] `backwards-compatibility-shim-migration-plan` — Since both `reasons/` and `reasons_lib/` exist side-by-side, understanding the migration state (which consumers still import the legacy path and when the shim can be removed) is critical for any refactor of `reasons.check_stale`.

## Beliefs

- `reasons-lib-check-stale-re-exports-reasons-check-stale` — This module forwards the entire public namespace of `reasons.check_stale` via a single `from reasons.check_stale import *`.
- `shim-adds-no-new-api` — The file defines no functions, classes, or constants of its own; its only runtime artifact is the imported namespace.
- `shim-requires-reasons-check-stale` — The shim has a one-way dependency on `reasons.check_stale`; any import-time failure in the canonical module propagates to legacy importers.
- `reasons-is-canonical-and-reasons-lib-is-legacy` — The docstring names `reasons.check_stale` as the intended import path, making `reasons_lib.check_stale` a deprecated alias.
- `shim-uses-noqa-f401-f403` — The star-import line carries `# noqa: F401,F403` to suppress unused-import and star-import lint warnings, and removing it would break lint checks without affecting runtime.
