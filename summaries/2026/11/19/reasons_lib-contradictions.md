---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/contradictions.py

# `reasons_lib/contradictions.py`

```python
"""Backwards-compatibility shim — use ``from reasons.contradictions`` instead."""
from reasons.contradictions import *  # noqa: F401,F403
```

## Purpose

This file owns **no logic**. It is a one-line compatibility alias that exists so that the legacy import path `reasons_lib.contradictions` keeps resolving to the real implementation, which now lives at `reasons.contradictions`.

The docstring is the contract: *"use `from reasons.contradictions` instead"*. The parallel `reasons/` and `reasons_lib/` trees in the repo tell the story — `reasons` is the canonical package and `reasons_lib` is the older / vendored-name package kept alive purely so existing call sites do not break. Every module in `reasons_lib/` that mirrors a `reasons/` module (api, ask, derive, storage, contradictions, …) follows this same shape.

## Key Components

There are **zero** classes, functions, or constants defined here. The entire public surface is whatever the star-import drags in.

- **Module docstring** — the only authored content; it documents the deprecation direction and is what `help(reasons_lib.contradictions)` will show (note: it is *not* the docstring of `reasons.contradictions`).
- **`from reasons.contradictions import *`** — rebinds every name in the target module's public namespace into this module's namespace. Because the statement is written with `*`, the set of names depends on whether `reasons/contradictions.py` defines `__all__`:
  - If it defines `__all__`, only those names are copied.
  - If not, every non-underscore name (including any helper/imported symbols the real module leaves at top level) is copied.

## Patterns

- **Backwards-compatibility shim / alias module.** Classic "move the code, keep the old import name" pattern. Deliberately minimal so there is no chance of the shim drifting out of sync with the implementation.
- **Wildcard re-export.** `import *` is normally discouraged, but here it is the right tool: the shim must expose *exactly* the target's public API without enumerating it (which would rot as the target evolves).
- **Suppressed linter noise.** `# noqa: F401,F403` is required because pyflakes/flake8 flag star imports as unused (`F401`) and star imports as forbidden (`F403`). The suppression is intentional and tightly scoped to this line.
- **Import-time only.** There is no runtime behavior, no function call, no laziness. All the work (and all failure modes) happen when the module is first imported.

## Dependencies

**Depends on:**
- `reasons.contradictions` — the real implementation. This is the *only* dependency; there is no stdlib or third-party use.

**Imported by (per repo context):**
- `reasons/api.py` — interesting: the canonical package imports the *shim*, not the implementation. This is a sign of an in-progress migration; it also means `reasons.api` transitively depends on `reasons.contradictions` through `reasons_lib.contradictions`, and any circular import there will surface as an import-time error in `reasons.api`.
- `tests/test_contradictions.py` — the test suite exercises the contradiction logic through the legacy path, which doubles as a regression test that the shim still re-exports everything the tests need.

## Flow

1. Something imports `reasons_lib.contradictions`.
2. Python imports and executes `reasons/contradictions.py` first (unless cached in `sys.modules`).
3. The star import copies the target's public names into the shim's module dict.
4. The shim module object is cached in `sys.modules['reasons_lib.contradictions']` and returned.

After step 3, `reasons_lib.contradictions.Foo is reasons.contradictions.Foo` for every exported `Foo` — the symbols are *shared* (same object identity), not copies. Mutation of a module-level constant or a class attribute through the legacy path is therefore visible through the new path and vice versa.

## Invariants

- **Name parity.** Whatever `reasons.contradictions` exports, `reasons_lib.contradictions` exports; nothing more is authored here.
- **No `__all__` on the shim.** The file does not define `__all__`, so `reasons_lib.contradictions.__all__` raises `AttributeError` even if `reasons.contradictions.__all__` exists. Any consumer that introspects `__all__` on the legacy module will break; consumers that do `from reasons_lib.contradictions import X` or `import *` work fine.
- **Docstring divergence.** `reasons_lib.contradictions.__doc__` is the shim text, not the target's docstring.
- **Object identity.** Exported objects are the same objects as the `reasons.*` ones; no wrapping, no proxying, no lazy re-binding.

## Error Handling

- **None authored.** There is no `try/except`, no validation, no fallback.
- **ImportError propagates.** If `reasons.contradictions` fails to import (missing optional dependency, syntax error, circular import with `reasons.api`), the failure surfaces as `ImportError` from `reasons_lib.contradictions` with the original traceback preserved in `__cause__`/chained context. A developer debugging a failure at this path should look *through* the shim straight at `reasons/contradictions.py` and its imports.
- **No silent degradation.** A name that disappears from `reasons.contradictions` becomes an `ImportError`/`AttributeError` at the legacy call site — which is the desired behavior because it forces the migration to the canonical path.

## Maintenance Guidance

- Do **not** add code here. Anything you need belongs in `reasons/contradictions.py`; adding it here means the two paths diverge.
- If you ever need to freeze the legacy surface (e.g., because `reasons.contradictions` started folding helpers into its namespace), add an explicit `__all__ = [...]` mirroring the intended public API, rather than relying on the implicit non-underscore rule.
- When `reasons/api.py` is updated to import from `reasons.contradictions` directly, this shim can eventually be deleted — but only after `tests/test_contradictions.py` and any external consumers are migrated.

## Topics to Explore

- [file] `reasons/contradictions.py` — The real implementation; inspect what it actually exports (and whether it defines `__all__`) to know the shim's implicit public surface.
- [file] `reasons_lib/__init__.py` — Whether the legacy package re-exports submodules eagerly or lazily, which affects whether importing `reasons_lib` triggers this shim at all.
- [file] `reasons/api.py` — Why the canonical package imports the legacy contradiction module instead of the canonical one; this is the key migration signal.
- [file] `tests/test_contradictions.py` — Confirms which symbols the suite expects to be reachable through the legacy path, i.e., the effective contract of this shim.
- [general] `reasons-vs-reasons_lib-migration` — Whether `pyproject.toml` ships both packages or aliases them, and whether there is a plan to retire `reasons_lib` entirely.

## Beliefs

- `reasons-lib-contradictions-is-a-compat-shim` — The module defines no logic and only re-exports `reasons.contradictions` via a star import.
- `reasons-lib-contradictions-requires-reasons-contradictions` — Importing the shim fails with the target module's `ImportError` if `reasons.contradictions` cannot be imported.
- `reasons-lib-contradictions-lacks-all` — The shim does not define `__all__`, so `reasons_lib.contradictions.__all__` raises `AttributeError` regardless of the target module.
- `reasons-lib-contradictions-exports-identical-objects` — Every exported symbol is the same object as its `reasons.contradictions` counterpart (`is` identity holds).
- `reasons-api-imports-legacy-contradictions` — `reasons/api.py` depends on `reasons_lib.contradictions` rather than importing `reasons.contradictions` directly, routing the canonical package through the shim.
