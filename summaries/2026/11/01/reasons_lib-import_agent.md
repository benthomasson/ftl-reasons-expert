---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/import_agent.py

# `reasons_lib/import_agent.py`

## Purpose

This is a **backwards-compatibility shim**. The project originally lived under the `reasons_lib` package name and was later renamed (or restructured) to `reasons`. Rather than breaking every downstream consumer that still does `from reasons_lib.import_agent import ...`, this file simply re-exports the entire public surface of the canonical module `reasons.import_agent`.

The docstring is explicit about the deprecation intent:

> *"Backwards-compatibility shim — use ``from reasons.import_agent`` instead."*

It owns **no logic of its own**. Its sole responsibility is to keep the old import path alive so that existing code (and the sibling modules `reasons_lib/api.py` and `reasons_lib/pg.py`) continues to resolve symbols without modification.

## Key Components

There are no classes, functions, or constants defined here. The entire module body is a single line:

```python
from reasons.import_agent import *  # noqa: F401,F403
```

| Element | Meaning |
|---|---|
| `import *` | Pulls every name in `reasons.import_agent`'s `__all__` (or, absent `__all__`, every public name not starting with `_`) into this module's namespace. |
| `# noqa: F401` | Suppresses the *pyflakes* "imported but unused" warning. The names are intentionally re-exported, not used locally. |
| `# noqa: F403` | Suppresses the *pyflakes* "star import" warning, acknowledging the deliberate wildcard. |

Whatever symbols a caller needs (likely things like an `import_agent` function or related constants/types) are resolved through this star-import at the `reasons_lib.import_agent` path.

## Patterns

- **Re-export / façade pattern.** The module acts as a thin alias for the canonical implementation. This is a standard technique during package renames or splits: you keep the old dotted path importable while nudging users toward the new one.
- **Star-import with lint suppression.** Because the file's *only* job is to forward names, a star import is the least-brittle option (it automatically tracks new exports added to `reasons.import_agent` without editing the shim). The `noqa` comments are the accepted trade-off.
- **Mirror-package layout.** Every file in `reasons/` has a same-named counterpart in `reasons_lib/`. This file is one instance of that systematic duplication-by-delegation.

## Dependencies

**Depends on (imports):**

- `reasons.import_agent` — the canonical module. This is the *only* real dependency. Everything else in the project is reachable transitively through whatever `reasons.import_agent` itself imports.

**Depended on (imported by):**

- `reasons_lib/api.py` — the sibling API module in the compat package; imports from `.import_agent` (or `reasons_lib.import_agent`) to access agent-import helpers.
- `reasons_lib/pg.py` — the sibling persistence module; similarly imports agent-related symbols.

> **Note:** If the "Imported By" entries genuinely point at `reasons/api.py` and `reasons/pg.py` (the *new* package), that would be an inverted dependency and worth flagging to the team. More likely these are the `reasons_lib/` siblings, and the listing simply abbreviated the path.

## Flow

Execution is trivial and happens **once, at import time**:

1. Python's import system encounters `reasons_lib.import_agent`.
2. It executes the single `from reasons.import_agent import *` statement.
3. Python imports `reasons.import_agent` (triggering *its* full import chain), inspects its namespace, and copies all public names into `reasons_lib.import_agent.__dict__`.
4. Any subsequent `from reasons_lib.import_agent import some_symbol` resolves against that copied namespace and returns the original object from `reasons.import_agent`.

There is no runtime branching, no data transformation, no side effects beyond the initial import of the target module.

## Invariants

- **Name parity is implicit, not enforced.** Because a star import is used, any *new* public symbol added to `reasons.import_agent` automatically appears in the shim. However, if a symbol is *removed* or *renamed* in `reasons.import_agent`, the shim will silently stop providing it — old callers will get an `ImportError` with no hint that the name moved.
- **No circular-import guard exists here.** The shim delegates to `reasons.import_agent`, which must not (directly or transitively) import back from `reasons_lib.import_agent`, or you get a circular import. The canonical module has no reason to do so, but it is an unenforced contract.
- **The shim is strictly read-only.** It defines no local state, so it cannot drift out of sync with the canonical module at runtime.

## Error Handling

This file **defines no error handling**. Two failure modes are possible, both inherited from Python's import machinery:

- **`ImportError` / `ModuleNotFoundError`** at import time if `reasons.import_agent` cannot be resolved (e.g., the `reasons` package isn't installed or isn't on `sys.path`). The `# noqa` comments have no effect here.
- **`ImportError`** at *call-site* import time if a consumer does `from reasons_lib.import_agent import SomeSymbol` and `SomeSymbol` no longer exists in `reasons.import_agent`. The error message will reference `reasons_lib.import_agent`, which could be slightly misleading since the symbol actually lives (or lived) in `reasons.import_agent`.

No exceptions are caught, logged, or re-raised.

---

## Topics to Explore

- [file] `reasons/import_agent.py` — The canonical implementation this shim forwards to; understanding its actual API surface (functions, dataclasses, constants) is essential to knowing what the star import exposes.
- [file] `reasons_lib/__init__.py` — Check whether the compat package's `__init__` also re-exports submodules, and whether it documents the deprecation strategy for the whole `reasons_lib` → `reasons` migration.
- [file] `reasons/api.py` — One of the listed importers; see how it references `import_agent` symbols and whether it already uses the `reasons.*` path or the legacy `reasons_lib.*` path.
- [general] `reasons-lib-to-reasons-migration` — The broader rename/restructure from `reasons_lib` to `reasons`. Understanding the migration plan (is it complete? is there a deprecation timeline?) helps decide whether these shims can be removed.
- [file] `tests/test_import_agent.py` — The test suite for the import-agent functionality; reading it reveals the expected public API and edge cases that the shim's star import must preserve.

## Beliefs

- `reasons-lib-import-agent-is-a-re-export-shim` — `reasons_lib/import_agent.py` defines no logic of its own; it exists solely to re-export the public namespace of `reasons.import_agent` for callers still using the old `reasons_lib` package path.
- `star-import-tracks-canonical-exports` — Because the shim uses `from reasons.import_agent import *`, any new public symbol added to `reasons.import_agent` is automatically visible through `reasons_lib.import_agent` without editing the shim, but a renamed or removed symbol will silently break old callers with a bare `ImportError`.
- `reasons-lib-import-agent-depends-on-reasons-import-agent` — The shim has exactly one real dependency: the canonical `reasons.import_agent` module. If that module's import chain changes, the shim's behaviour changes with it.
- `noqa-suppresses-f401-and-f403` — The `# noqa: F401,F403` comment suppresses both "imported but unused" (F401) and "star import" (F403) linter warnings, indicating the wildcard re-export is intentional and the names are meant to be consumed by downstream importers, not used within this file.
