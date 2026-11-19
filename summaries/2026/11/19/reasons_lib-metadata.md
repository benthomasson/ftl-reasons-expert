---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/metadata.py

# `reasons_lib/metadata.py`

```python
"""Backwards-compatibility shim — use ``from reasons.metadata`` instead."""
from reasons.metadata import *  # noqa: F401,F403
```

That is the entire file. Everything substantive lives in the canonical module `reasons/metadata.py`; this file exists purely to keep the old import path working during a package rename.

## Purpose

The repo ships **two parallel package trees** with identical module names:

- `reasons/` — the *canonical* package.
- `reasons_lib/` — a legacy / alternate import path kept alive for backward compatibility.

`reasons_lib/metadata.py` is a **re-export shim**: any code that still does `from reasons_lib.metadata import …` (or `import reasons_lib.metadata`) continues to work because every public name from `reasons.metadata` is pulled into this module's namespace via the wildcard import. The module docstring explicitly documents the intended replacement path — `from reasons.metadata import ...`.

It owns **no logic, no state, and no side effects** beyond the single import. Its sole responsibility is namespace aliasing so that a rename (`reasons_lib` → `reasons`, or vice versa) can be rolled out without a flag-day migration of every caller.

## Key Components

There are no classes, functions, or constants defined here. The only component is:

| Element | Contract |
|---|---|
| `from reasons.metadata import *` | Imports every name honored by `reasons.metadata`'s export surface into this module's globals. If `reasons.metadata` defines `__all__`, only those names flow through; otherwise every non-underscore name is imported. |
| `# noqa: F401,F403` | Silences flake8/ruff — `F401` ("imported but unused") and `F403` ("`from module import *` used, unable to detect undefined names") — both of which are unavoidable and correct in a shim like this. |
| Module docstring | Declares the canonical replacement (`reasons.metadata`) so a maintainer reading the file knows where the real implementation lives. |

Public names exported by this module therefore mirror `reasons.metadata` — e.g. any helpers it exposes for tags, provenance, `defeater` metadata, `source_type`, `pin` information, etc. Consumers should not need to know they're hitting the shim.

## Patterns

1. **Backwards-compatibility shim / facade.** A one-line module whose body aliases another module's namespace. Common when a library is renamed but the old import path must keep working (e.g. during a deprecation window). The parallel `reasons/` vs `reasons_lib/` trees in the repo listing strongly suggest this is a rename-in-progress: every module in `reasons_lib/` likely has the same `from reasons.<name> import *` shape.
2. **Wildcard re-export with `noqa`.** The idiomatic Python way to make `A` and `B` the *same* public API without enumerating every symbol by hand. Trade-off: it's fragile to linting and introspection (see "Invariants").
3. **Docstring-as-deprecation-notice.** Rather than emitting a `DeprecationWarning`, the shim documents intent in the docstring. That means callers get zero runtime signal — the deprecation is a *social* contract, not a technical one.

## Dependencies

**Imports (this file →):**
- `reasons.metadata` — the sole dependency. If that module fails to import (missing optional deps, circular import during package init, etc.), `import reasons_lib.metadata` will fail identically.

**Imported by (→ this file):**
- The tool's dependency graph lists `reasons/api.py`, `reasons/cli.py`, `reasons/export_card.py`, `reasons/export_markdown.py`, `reasons/metadata.py`, `reasons/pg.py`, `reasons/storage.py`, and tests (`test_metadata.py`, `test_metadata_extended.py`, `test_export_card.py`). Note: because the canonical module and the shim share the symbol name `metadata`, the graph likely conflates the two — the primary consumers almost certainly import `from reasons.metadata import ...`, and this shim exists so *legacy/external* callers (or path-dependent imports) that say `reasons_lib.metadata` still resolve. A maintainer should grep for actual `from reasons_lib.metadata` usage before assuming any in-tree consumer depends on the shim specifically.

The architectural dependency rule is one-directional: **`reasons_lib` depends on `reasons`, never the reverse.** If `reasons.metadata` ever imports `reasons_lib.metadata` back, you get a circular import — the shim must be a leaf.

## Flow

Trivial, but ordered:

1. Python executes `import reasons_lib.metadata`.
2. The interpreter imports `reasons` (package `__init__.py` runs first — note this means any side effects in `reasons/__init__.py` fire when the *shim* is imported, which can surprise people expecting `reasons_lib` to be standalone).
3. The interpreter imports `reasons.metadata` in full.
4. The wildcard import iterates `reasons.metadata.__all__` (if defined) or `dir(reasons.metadata)` else, and binds each name into `reasons_lib.metadata`'s globals.
5. The shim module object is cached in `sys.modules['reasons_lib.metadata']`, and `sys.modules['reasons_lib.metadata']` is a *distinct module object* from `sys.modules['reasons.metadata']` — they merely share name bindings.

## Invariants

- **Only public names propagate.** Underscore-prefixed names in `reasons.metadata` (e.g. private helpers like `_normalize_tag`) are *not* imported by `import *`. If any legacy caller relied on `reasons_lib.metadata._foo`, this shim breaks them.
- **`__all__` is honored, not copied.** If `reasons.metadata.__all__` is defined, `import *` respects it — so `reasons_lib.metadata` will contain exactly those names. But the shim does **not** define its own `__all__`, so a subsequent `from reasons_lib.metadata import *` re-exports every non-underscore name *present in the shim*, which can diverge from `reasons.metadata.__all__` (e.g. if `__all__` lists an underscore name, or omits a non-underscore name that was still imported). This is the subtle wildcard-of-wildcard gap to watch for.
- **Identity is not preserved for rebindings.** `reasons_lib.metadata.SomeClass is reasons.metadata.SomeClass` holds for classes/functions (same object), but reassigning `reasons_lib.metadata.FOO = …` does **not** affect `reasons.metadata.FOO`, and vice versa. Monkeypatches through the shim won't be visible to the canonical module.
- **No `sys.modules` aliasing.** A stronger shim would do `sys.modules['reasons_lib.metadata'] = reasons.metadata` so both names resolve to the *same* module object. This file does not — it creates a separate module object whose bindings are copies. Code that does `isinstance(x, reasons_lib.metadata.SomeClass)` still works (class objects are shared), but code that inspects `module.__dict__` or sets module-level state will see divergence.
- **Type-checker and introspection limits.** Static analyzers and `dir(reasons_lib.metadata)` generally do not fully follow wildcard re-exports across packages; IDE autocompletion on the shim may be worse than on the canonical module. That's another reason the docstring redirects maintainers to `reasons.metadata`.

## Error Handling

- **No error handling.** No `try/except`, no `warnings.warn`, no `DeprecationWarning`. Any failure importing `reasons.metadata` (e.g. a missing optional dependency, circular import, or syntax error in the real module) propagates unchanged out of `import reasons_lib.metadata`.
- **No deprecation signal.** Callers using `reasons_lib.metadata` receive exactly the same behavior and stack traces as callers using `reasons.metadata`. The only way a developer learns they're on the shim is by reading this file or profiling imports — which means dead shims tend to live longer than planned. If you need to force migration, add a `warnings.warn(..., DeprecationWarning, stacklevel=2)` here in a future change.
- **`# noqa` hides the honest noise.** `F401`/`F403` are precisely the diagnostics that would flag this shim as suspicious. Suppressing them is correct, but do not "fix" the suppression by removing it — you'd just be re-adding lint failures for a legitimate pattern.

## Topics to Explore

- [file] `reasons/metadata.py` — The real implementation; this is where all tag, provenance, defeater, and source-type logic actually lives, and its `__all__` (if any) defines what this shim exposes.
- [general] `reasons-vs-reasons_lib-rename` — Determine which tree is canonical, whether `reasons_lib` is deprecated upstream or downstream, and whether a `DeprecationWarning` rollout / removal timeline exists (check `CHANGELOG.md` and `pyproject.toml`).
- [general] `import-star-reexport-hazards` — The `__all__` interaction, `sys.modules` identity divergence, and monkeypatch-visibility caveats described in the Invariants section; decide whether to upgrade the shim to `sys.modules[...] = ...` aliasing.
- [file] `reasons_lib/__init__.py` — Confirm whether the package `__init__.py` also re-exports from `reasons`, or whether the shims are strictly per-module.
- [file] `tests/test_metadata_extended.py` — See which module path the tests import; if they import `reasons.metadata` directly, the shim is untested and can silently rot.

## Beliefs

- `reasons-lib-metadata-is-a-bc-shim` — `reasons_lib/metadata.py` contains no implementation and exists solely to re-export the public API of `reasons.metadata` for backward compatibility.
- `reasons-metadata-is-canonical` — The module docstring designates `reasons.metadata` as the canonical import path; `reasons_lib.metadata` is the compatibility alias.
- `reasons-lib-metadata-imports-from-reasons-metadata` — The single statement `from reasons.metadata import *` creates a one-way compile-time dependency from `reasons_lib` to `reasons`, so `reasons.metadata` must not import `reasons_lib.metadata` (would be circular).
- `wildcard-reexport-honors-__all__` — Only names in `reasons.metadata.__all__` (or all non-underscore names if `__all__` is absent) become attributes of `reasons_lib.metadata`; underscore-prefixed names are not re-exported.
- `shim-emits-no-deprecation-warning` — The shim contains no `DeprecationWarning`; using `reasons_lib.metadata` produces no runtime signal that a migration is expected.
- `noqa-suppresses-f401-and-f403` — The trailing `# noqa: F401,F403` suppresses "unused import" and "wildcard import" lint diagnostics, both of which are intentional for this shim pattern.
