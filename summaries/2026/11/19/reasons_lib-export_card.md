---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/export_card.py

## Overview

`reasons_lib/export_card.py` is a **backwards-compatibility shim** — a two-line facade that re-exports the public surface of `reasons.export_card`. It contains no application logic; its sole responsibility is to keep the historical import path `from reasons_lib.export_card import ...` working after the codebase was reorganized so that `reasons.export_card` is the canonical home for the card-exporting implementation.

```python
"""Backwards-compatibility shim — use ``from reasons.export_card`` instead."""
from reasons.export_card import *  # noqa: F401,F403
```

The repository layout confirms the pattern: `reasons/` and `reasons_lib/` contain an almost identical list of modules (`api.py`, `ask.py`, `derive.py`, `export_card.py`, `export_markdown.py`, …). Every non-shim file in `reasons_lib/` presumably mirrors a `reasons/` counterpart; this one is the thinnest possible mirror.

---

## 1. Purpose

- **Migration bridge.** Consumers (internal modules like `reasons/api.py`, and external code that still `import reasons_lib.export_card`) continue to work after the implementation moved under `reasons/`.
- **Single source of truth.** Any behavior, signatures, exceptions, and side effects of exporting a card live exclusively in `reasons/export_card.py`. This file intentionally owns no semantics of its own — changing it should never change what card-export does.
- **Deprecation boundary.** The docstring is user-facing documentation that directs new code to `reasons.export_card`, marking `reasons_lib.export_card` as legacy.

## 2. Key Components

There are none in the usual sense — no classes, functions, constants, or module-level state are defined **here**. The only "component" is the wildcard import statement, whose contract is:

| Aspect | Contract |
|---|---|
| Import target | `reasons.export_card` must exist and be importable |
| Bindings | Every name reachable from `reasons.export_card` that is **not** prefixed with `_` and is either listed in its `__all__` or (absent `__all__`) not underscore-prefixed, is bound into this module's global namespace at import time |
| Own `__all__` | Not defined here, so `from reasons_lib.export_card import *` behaves like re-exporting the same public names (Python's `import *` on a module without `__all__` exposes only non-underscore names) |
| Own `__doc__` | Set to the shim directive string above |

Practical consequences:
- `reasons.export_card.SomePublicClass` and `reasons_lib.export_card.SomePublicClass` resolve to **the same object** (identity preserved — no copies, no wrappers).
- If `reasons.export_card` later renames a symbol, both paths break together — this is by design (single-source, no drift).

## 3. Patterns

- **Facade / Compatibility Shim.** Classic two-line re-export shim; occasionally called the "alias module" pattern.
- **Wildcard re-export (`from X import *`).** Preferred here over `from X import a, b, c` because the shim should track whatever public surface `reasons.export_card` exposes without needing to be edited when that surface changes. The tradeoff is that static analyzers can't resolve names through this module — hence the `noqa`.
- **`# noqa: F401,F403`.** `F401` = "imported but unused" (the imported names *are* the product), `F403` = "`import *` used; unable to detect undefined names". This is the idiomatic lint-suppression pair for a re-export module.
- **Convention: legacy package = new package.** The parallel `reasons/` vs `reasons_lib/` trees are the repository-wide expression of the same pattern, applied per-module.

## 4. Dependencies

**Depends on (runtime):**
- `reasons.export_card` — the canonical module. Importing `reasons_lib.export_card` *triggers the full initialization* of `reasons.export_card` (its own imports, module-level constants, etc.), so any transitive dependency of `reasons.export_card` (e.g. survey/transport clients, LLM helpers, or whatever it uses) is pulled in transitively.

**Depends on (tooling):**
- `ruff`/`flake8` honoring `# noqa` codes; a linter that flags `import *` without honoring `noqa` would reject this file.

**Imported by:**
- `reasons/api.py` — a module in the *new* namespace still imports via `reasons_lib.export_card` for at least part of its surface. This is the interesting coupling: even the canonical API module may piggy-back on the shim, meaning removal of this shim would break `reasons/api.py` unless that import is migrated first.
- `tests/test_export_card.py` — exercises the shim so the legacy path stays tested, and likely also cross-checks that names exported here are identical to `reasons.export_card`.

Note the dependency direction: `reasons_lib` → `reasons`, never the reverse (except for `reasons/api.py` reaching through the shim). That direction must be preserved — if `reasons.export_card` ever imported from `reasons_lib.export_card`, you'd get a circular import.

## 5. Flow

At import time, executing `import reasons_lib.export_card` (or `from reasons_lib.export_card import foo`) runs exactly:

1. Import machinery loads `reasons.export_card` (running its module-level code the first time; cached in `sys.modules` subsequently).
2. `import *` enumerates the names to export: `__all__` if defined by `reasons.export_card`, otherwise all attributes of that module not starting with `_`.
3. Those names are assigned into the current module's `globals()`.
4. Control returns to the caller, which may now access `reasons_lib.export_card.foo`.

No transformations, no validation, no side effects beyond the transitive import of `reasons.export_card`. Because `import *` **rebinds existing names**, if some future version of this file defined a local `foo` and then added the wildcard import below it, the wildcard would shadow the local. Today the file has zero such collisions — keep it that way.

## 6. Invariants

- **Import-order invariant:** `reasons.export_card` must be importable and free of import-time errors. This shim cannot mask an `ImportError` from the canonical module.
- **Identity invariant:** for every exported name `n`, `reasons_lib.export_card.n is reasons.export_card.n`. The shim must never wrap, copy, or subclass.
- **Namespace-purity invariant:** this module defines no names of its own (other than `__doc__` and the `reasons` reference created by the import statement). Adding a helper here would risk silently shadowing a canonical export.
- **Direction invariant:** the shim imports the canonical module; the canonical module and `reasons/` never import the reverse (except `reasons/api.py` accessing the shim).
- **Public-surface invariant:** names beginning with `_` and names excluded from `reasons.export_card.__all__` are **not** re-exported. If a caller needs one of those, they must use `reasons.export_card` directly — the shim's visibility is strictly the public surface.

## 7. Error Handling

The shim itself contains no `try`/`except` and no validation. Errors surface as normal Python import failures:

- **`ModuleNotFoundError: No module named 'reasons.export_card'`** — raised if the canonical module is renamed or removed. This propagates unchanged to whatever imported the shim (e.g. `reasons/api.py`).
- **Errors raised while executing `reasons.export_card`'s module body** (e.g. a config error, missing optional dependency) propagate through this module and prevent it from completing initialization, exactly as if the caller had imported the canonical module directly. There is no error swallowing.
- **`ImportError: cannot import name 'X'`** — raised by callers doing `from reasons_lib.export_card import X` when `X` is not in the canonical module's public surface. This is the correct failure mode: it tells the developer the symbol has moved or was never public.
- **Lint errors** (F401/F403) are suppressed via `# noqa`; the codes are deliberate, not accidental.

---

## Topics to Explore

- [file] `reasons/export_card.py` — The canonical implementation this shim re-exports; everything meaningful about card export lives here, not in `reasons_lib/export_card.py`.
- [file] `reasons_lib/__init__.py` — Shows how the legacy package is packaged and whether it re-exports submodules or performs any migration/deprecation wiring at the package level.
- [file] `tests/test_export_card.py` — Documents the contract, and likely tests both import paths and identity-equality between `reasons.export_card` and `reasons_lib.export_card` symbols.
- [file] `reasons/api.py` — Demonstrates that the new code path still imports through the legacy shim; useful to know before attempting to retire `reasons_lib`.
- [general] `compatibility-shim-pattern` — The broader migration story: how the repository plans to retire `reasons_lib/` and whether there is a deprecation timeline (check `CHANGELOG.md` and `CLAUDE.md`).

## Beliefs

- `reasons-lib-export-card-is-a-shim` — `reasons_lib/export_card.py` contains no implementation logic; its only statement is `from reasons.export_card import *`.
- `reasons-lib-export-card-requires-reasons-export-card` — Importing this module raises `ModuleNotFoundError` if `reasons.export_card` cannot be imported.
- `reasons-lib-export-card-reexports-only-public-names` — Only names in `reasons.export_card.__all__` (or, if unset, non-underscore names) are exposed through this shim; symbols with leading underscores are not re-exported.
- `reasons-lib-export-card-preserves-identity` — Re-exported objects are the same Python objects as their `reasons.export_card` counterparts, not copies or wrappers.
- `reasons-lib-export-card-has-two-consumers` — `reasons/api.py` and `tests/test_export_card.py` both import through this path, so removing the shim requires migrating both.
