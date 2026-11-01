---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/derive.py

# `reasons_lib/derive.py` — Backwards-Compatibility Shim

## Purpose

This file is a **one-line re-export shim**. Its sole job is to keep the old import path `reasons_lib.derive` working so that any existing consumer (scripts, notebooks, downstream projects) that wrote `from reasons_lib.derive import derive` continues to resolve without modification. The docstring makes the migration target explicit: code should be updated to `from reasons.derive import …` instead.

In the broader project layout you can see two parallel packages:

| Package | Role |
|---|---|
| `reasons/` | **Canonical** implementation (contains `derive.py` with the actual logic) |
| `reasons_lib/` | **Legacy alias layer** — every module here is a thin `from reasons.X import *` re-export |

This is a classic package-rename migration strategy: ship the new name, keep the old name as a shim, and let consumers migrate at their own pace.

## Key Components

There is exactly **one statement** in the file:

```python
from reasons.derive import *  # noqa: F401,F403
```

- **`import *`** — pulls every public name (anything not starting with `_`) from `reasons.derive` into the `reasons_lib.derive` namespace. Whatever functions, classes, or constants `reasons.derive` exports (e.g., a `derive` function, budget constants, dataclasses) are all visible here.
- **`# noqa: F401`** — silences the linter's "imported but unused" warning. The import *is* used; it's the entire point of the module.
- **`# noqa: F403`** — silences the "star-notation import" warning, which is typically discouraged because it makes the public surface opaque. Here it's intentional: the shim should track the upstream surface automatically.

There are no local definitions, no `__all__`, no type stubs. The module's `__all__` (if it exists) is inherited from whatever `reasons.derive` defines.

## Patterns

- **Shim / facade for a package rename.** The repo has a 1-to-1 mapping between `reasons/<module>.py` and `reasons_lib/<module>.py`. Every `reasons_lib` file follows the same two-line pattern (docstring + star import). This keeps the migration reversible and zero-cost for existing importers.
- **`noqa` pragmas as documentation.** The two suppression codes double as a note to future maintainers: "yes, we know this looks like a smell; it's deliberate."
- **No `__init__.py` logic here.** Package-level exports are governed by `reasons_lib/__init__.py` (not shown), not by individual shim files.

## Dependencies

**Upstream (this file imports):**

- `reasons.derive` — the real implementation. If you remove or rename a symbol there, every star-import consumer (including this shim) silently loses it. There is no compile-time check that the surfaces match.

**Downstream (imported by this file):**

| Importer | Why it matters |
|---|---|
| `reasons/api.py` | Public HTTP/API layer that calls derive logic |
| `reasons/cli.py` | CLI entry point; exposes `derive` as a subcommand |
| `tests/test_derive.py` | Core unit tests for the derive algorithm |
| `tests/test_derive_budget.py` | Budget/cost-constraint tests |
| `tests/test_source_type.py` | Tests that derive respects source-type metadata |

Note that `reasons/api.py` and `reasons/cli.py` are in the *canonical* package yet are listed as importers of the *shim* package. This likely means they do `from reasons_lib.derive import …` for historical reasons and have not yet been updated to import from `reasons.derive` directly. That's the exact technical debt this shim is papering over.

## Flow

Execution is trivial:

1. Python resolves `reasons_lib/derive.py`.
2. The module docstring is evaluated (no-op).
3. `from reasons.derive import *` triggers execution of `reasons/derive.py`, importing all its public names.
4. Those names are injected into `reasons_lib.derive.__dict__`.
5. Any subsequent `from reasons_lib.derive import X` simply does a dict lookup in step 4.

There is no branching, no state, no side effects beyond what `reasons.derive` itself does at import time.

## Invariants

- **Surface equivalence (fragile):** `set(dir(reasons_lib.derive)) ⊇ set(dir(reasons.derive))`. Because the shim uses `*` and there is no explicit `__all__` here, this holds *as long as* `reasons.derive` doesn't define a private `__all__` that excludes names the old API exposed. If someone adds `__all__ = ["derive"]` in `reasons.derive`, the shim silently drops everything else.
- **Single source of truth:** No logic lives in `reasons_lib/`. All behaviour is owned by `reasons/`. The shim adds no validation, no wrapping, no deprecation warning.
- **Import-order safety:** `reasons` must be importable before `reasons_lib` is loaded. Since both live in the same project, this is guaranteed as long as the package is installed (see `pyproject.toml`).

## Error Handling

There is none here. Errors surface in two ways:

1. **Import-time:** If `reasons.derive` doesn't exist or fails to import (e.g., a missing dependency like `llm.py` raising at module level), the `from reasons.derive import *` line raises `ImportError` or whatever the underlying exception is. The shim does not catch or re-raise with context.
2. **Attribute-time:** If a consumer does `from reasons_lib.derive import some_function` and that name was removed from `reasons.derive`, Python raises `ImportError: cannot import name 'some_function' from 'reasons_lib.derive'`. The error message will point at the *shim* path, which can be mildly confusing.

No try/except, no logging, no `DeprecationWarning` is emitted. A developer migrating imports gets zero signal that the old path is being phased out.

## Topics to Explore

- [file] `reasons/derive.py` — The actual derivation logic (budget constraints, premise resolution, source-type handling) that this shim re-exports
- [file] `reasons_lib/__init__.py` — Whether the package-level `__init__` adds a `DeprecationWarning` or `__all__` that gates which shims are visible
- [file] `tests/test_derive.py` — How the derive algorithm is tested end-to-end; reveals the public API surface that the shim must preserve
- [general] `reasons-vs-reasons-lib-migration` — Check `git log` / `CHANGELOG.md` for when `reasons_lib` was introduced and whether there's a plan to delete it; determines whether this shim is temporary or permanent
- [diff] `reasons/api.py` — Whether it imports via `reasons_lib.derive` or `reasons.derive`; if the former, it's a candidate for updating to the canonical path and eliminating one shim dependency

## Beliefs

- `reasons-lib-replaces-reasons-import-path` — `reasons_lib/<module>.py` is a drop-in alias for `reasons/<module>.py`; the two packages expose an identical public surface for every module.
- `star-import-shim-preserves-surface-dynamically` — Because the shim uses `import *` with no local `__all__`, adding a new public symbol to `reasons.derive` automatically makes it available via `reasons_lib.derive` with zero code changes in the shim.
- `reasons-lib-shim-emits-no-deprecation-warning` — Importing from `reasons_lib.derive` produces no `DeprecationWarning` or stderr message; consumers are unaware the path is slated for removal.
- `reasons-api-and-cli-depend-on-shim` — `reasons/api.py` and `reasons/cli.py` (canonical package) currently import derive through `reasons_lib.derive`, creating an intra-project dependency on the legacy alias layer.
- `noqa-F403-suppresses-star-import-lint` — The `# noqa: F403` comment disables flake8's "star-notation import" rule for this file, meaning the linter will not flag the `import *` line even under a strict configuration.
