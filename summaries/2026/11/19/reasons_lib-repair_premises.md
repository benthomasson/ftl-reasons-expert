---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/repair_premises.py

# `reasons_lib/repair_premises.py`

```python
"""Backwards-compatibility shim — use ``from reasons.repair_premises`` instead."""
from reasons.repair_premises import *  # noqa: F401,F403
```

That is the entire file — two lines. Everything interesting about it is *why* it exists and what it implies for maintainers.

---

## 1. Purpose

This module is a **deprecation/alias shim**. It owns no logic. Its sole responsibility is to make the legacy import path

```python
from reasons_lib.repair_premises import repair_premises
```

keep working after the project renamed its import package from `reasons_lib` to `reasons`.

You can see the rename in the repository layout: the tree contains **two sibling packages with parallel contents** (`reasons/` and `reasons_lib/`), each with the same ~30 module names. The `reasons/` package is canonical; the `reasons_lib/` package is the legacy namespace retained so downstream code (and `tests/test_repair_premises.py`) does not break. This file is the `repair_premises` member of that legacy surface.

The docstring states the contract explicitly: *"use `from reasons.repair_premises` instead."* Anything new should import from `reasons`, not from here.

## 2. Key Components

There are **no classes, functions, or constants defined here.** The module's only symbol-producing statement is a star import, so the effective public API is:

> *Every name that `reasons.repair_premises` exposes via star-import becomes a module-level attribute of `reasons_lib.repair_premises`.*

Concretely, the names you get are determined by the source module:

- If `reasons/repair_premises.py` defines `__all__`, exactly those names are re-exported.
- If it does not, star import re-exports all module-level names **that do not begin with an underscore** — including names imported into it (e.g., `dataclass`, `json`, `Path`), unless guarded by `__all__`.

The module has no `__all__` of its own, so a second-order `from reasons_lib.repair_premises import *` behaves identically to importing the source module directly.

The two lint suppressions are load-bearing documentation, not noise:

| Code | Meaning | Why suppressed |
|---|---|---|
| `F401` | imported but unused | the import exists *for* the re-export side effect |
| `F403` | `import *` used | intentional wildcard forwarding |

## 3. Patterns

**Backwards-compatibility shim / namespace alias.** The standard idiom for an import-path rename when you cannot (yet) do a clean break:

```python
"""Backwards-compatibility shim — use ``from <new.path>`` instead."""
from <new.path> import *  # noqa: F401,F403
```

Related idioms and what this file deliberately does *not* do:

- **No `DeprecationWarning` emission.** A more thorough shim would call `warnings.warn(..., DeprecationWarning, stacklevel=2)` at import time. This one is silent, so the legacy path produces no runtime signal that it is obsolete. Discovery of stale call sites is therefore a static-analysis / grep problem, not a runtime one.
- **No `__all__` mirroring and no explicit symbol re-binding.** `from x import name` would give you a real attribute rebinding and keep `reasons_lib.repair_premises.foo is reasons.repair_premises.foo`; star import achieves the same identity for the forwarded names, but only for names that pass the star-import filter.
- **Wildcard forwarding** rather than curated exports — meaning the shim's surface grows automatically if the source module adds public names. That is the intended trade-off: zero maintenance, at the cost of over-exposing internals (see §6).

## 4. Dependencies

**Depends on (imports):**

- `reasons.repair_premises` — the real implementation. This is the only dependency, and it is a hard one: the shim cannot function unless the `reasons` package is importable.

**Depended on by (imported_by):** per the repository context,

- `tests/test_repair_premises.py` — presumably exercises the legacy import path (directly, or transitively via fixtures).
- `reasons/api.py` — the *canonical* package importing from the *legacy* package. That direction is worth noting: it means `reasons.api` → `reasons_lib.repair_premises` → `reasons.repair_premises`. This inverts the usual "new code never imports old code" rule and is a latent circular-import hazard if `reasons/repair_premises.py` ever imports `reasons.api` (or anything that imports `reasons.api`) at module top level.

Note the asymmetry: `reasons_lib/repair_premises.py` imports from `reasons/`, never the reverse *within this module* — but `reasons/api.py` importing this shim creates a back-edge at the package level.

## 5. Flow

Import-time only. There is no function call, no state, no runtime branch.

1. Something does `import reasons_lib.repair_premises` (or `from reasons_lib.repair_premises import X`).
2. Python's import machinery executes this module body:
   - Checks `sys.modules` for `reasons.repair_premises`. If present, reuses it; otherwise imports it recursively — running the **entire** `reasons/repair_premises.py` top to bottom, including any module-level work (constants, dataclass definitions, regex compilation, potentially a storage backend handle).
   - Binds each star-exported name into the `reasons_lib.repair_premises` module namespace.
3. The module object is cached in `sys.modules["reasons_lib.repair_premises"]`.

**Consequence for identity:** forwarded *values* are identical objects — `reasons_lib.repair_premises.SOME_CONST is reasons.repair_premises.SOME_CONST` is `True`, and a forwarded class is the *same* class object, so `isinstance` checks interoperate across both import paths.

**Consequence for monkeypatching:** the two modules are **distinct module objects**. `monkeypatch.setattr("reasons_lib.repair_premises.foo", fake)` mutates only the `reasons_lib` namespace; any code that calls `reasons.repair_premises.foo` internally still sees the original. Since rebinding in the source module is what actually affects behaviour, tests must patch the **origin** module, not the shim — or patch through the caller's namespace. This is the number-one gotcha for anyone writing tests against this file.

## 6. Invariants

- **No logic may be added here.** Any behaviour implemented in this file exists twice (once here, once in `reasons/`), so the two drift. The file must remain a pure re-export.
- **The import graph must be acyclic at module-load time.** `reasons.repair_premises` must not transitively import `reasons_lib.repair_premises` at top level, because the shim's very first statement triggers the full source-module load.
- **The `reasons` package must be importable whenever `reasons_lib` is.** This shim is not self-contained: shipping `reasons_lib` without `reasons` turns every import of this module into an `ImportError`. The packaging config (`pyproject.toml`) must include both packages.
- **Public surface is a superset, not an exact match, of the source's intended API** when the source lacks `__all__` — star import leaks the source module's own imports (helpers, stdlib names, dataclasses, type aliases) into the legacy namespace. Callers may have (accidentally) come to rely on these, freezing them into the de-facto contract.
- **Underscore-private names are not forwarded.** `_helper` in `reasons/repair_premises.py` is invisible here even without `__all__`. If any legacy consumer touches a private name, that consumer is already broken and the failure is a clean `AttributeError`.
- **Introspection metadata is not mirrored.** `reasons_lib.repair_premises.__doc__` is the shim docstring; `__file__`, `__name__`, and `__spec__` point at the shim. Only `__all__` (if the source defines one) travels across.

## 7. Error Handling

The file contains **no `try`/`except` and raises nothing of its own.** All error semantics are inherited from the import system:

- **`ImportError` / `ModuleNotFoundError`** if `reasons` is not installed or not on `sys.path`. The traceback will surface `reasons_lib/repair_premises.py, line 2` as the immediate origin, which can be misleading when the real problem is a packaging omission in `pyproject.toml` (`packages` / `include` lists). When you see this error, check distribution metadata before checking this file.
- **`ImportError` from a partially initialized module** if a circular import is introduced upstream — the classic symptom is `ImportError: cannot import name 'X' from partially initialized module 'reasons.repair_premises' (most likely due to a circular import)`. That is the failure mode the §6 acyclicity invariant guards against.
- **Any exception raised at import time by `reasons/repair_premises.py`** (bad config, missing optional dependency, failed DB connection at module scope) propagates verbatim through this shim, with one extra frame in the traceback.
- **`AttributeError`** on access to a name excluded by the star-import filter (underscore-prefixed, or absent from `__all__`). This is the intended, non-silent failure for consumers reaching beyond the forwarded surface.

The shim deliberately **does not swallow** the source module's import-time failures — good, because silently degrading would hide real breakage.

## Topics to Explore

- [file] `reasons/repair_premises.py` — The real implementation this shim forwards; read it to enumerate the exact forwarded API surface and check whether it defines `__all__`.
- [file] `reasons/api.py` — Why the canonical package imports the legacy shim, and whether that edge creates a circular-import risk.
- [file] `pyproject.toml` — Packaging configuration must list both `reasons` and `reasons_lib`; a missing entry is the most likely cause of `ImportError` here.
- [function] `tests/test_repair_premises.py` — Confirm whether tests patch names via `reasons_lib.…` (which would be a no-op against internal callers) or via the origin module.
- [general] `namespace-rename-migration` — How widely `reasons_lib` is still referenced across the repo, and what a removal plan (deprecation warning → grep → delete) would look like.

## Beliefs

- `reasons-lib-repair-premises-is-a-shim` — The module defines no functions or classes; its entire body is a docstring plus one star import from `reasons.repair_premises`.
- `reasons-lib-repair-premises-requires-reasons-package` — Importing this module fails with `ModuleNotFoundError` unless the sibling `reasons` package is importable.
- `star-import-does-not-forward-private-names` — Names beginning with `_` in `reasons/repair_premises.py` are not accessible via `reasons_lib.repair_premises`.
- `shim-and-origin-are-distinct-module-objects` — Monkeypatching `reasons_lib.repair_premises.<name>` does not affect `reasons.repair_premises.<name>`, so tests must patch the origin module.
- `reasons-api-imports-legacy-shim` — `reasons/api.py` depends on the legacy namespace, creating a back-edge from the canonical package to the compatibility layer.
- `shim-emits-no-deprecation-warning` — Using the legacy import path produces no runtime deprecation signal; stale call sites must be found statically.
