---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/cluster.py

# `reasons_lib/cluster.py` — Compatibility Shim

## Purpose

This file is a **one-line backwards-compatibility shim**, not a real implementation. The entire file is:

```python
"""Backwards-compatibility shim — use ``from reasons.cluster`` instead."""
from reasons.cluster import *  # noqa: F401,F403
```

Its only job is to keep old import paths working after the package was reorganized: any code that still does `from reasons_lib.cluster import X` continues to work because the shim re-exports every public name from `reasons.cluster`. The docstring explicitly steers new code toward `from reasons.cluster import ...` instead.

The **actual clustering logic lives in `reasons/cluster.py`** (the `reasons/` package is the modern canonical home; `reasons_lib/` mirrors it directory-for-directory, e.g., `api.py`, `cli.py`, `derive.py`, `contradictions.py`, etc.). This shim adds no behavior, state, or constants of its own.

## Key Components

There are no classes, functions, or module-level constants defined here. The single significant construct is the **wildcard import**:

| Component | Contract |
|---|---|
| `from reasons.cluster import *` | Binds every name in `reasons.cluster.__all__` (or every non-underscore-prefixed name if `__all__` is absent) into the `reasons_lib.cluster` module namespace. |
| `# noqa: F401,F403` | Suppresses flake8 for (a) the wildcard import being "unused" and (b) the use of a wildcard import. This is the standard idiom for re-export shims. |

Because it uses `*`, the shim **inherits the public API of `reasons.cluster` verbatim** — including any private-by-convention names that lack a leading underscore but are technically public. What `X` in `from reasons_lib.cluster import X` succeeds for is entirely determined by `reasons/cluster.py`'s public surface.

## Patterns

1. **Facade / compatibility shim.** Classic pattern for a package rename (`reasons_lib` → `reasons`) that must not break downstream importers. The `reasons_lib/` tree and `reasons/` tree are near-identical mirrors; each `reasons_lib/*.py` file is presumably a shim.
2. **Wildcard re-export.** `from module import *` is used here as a deliberate "pass-through": the shim is a transparent alias, not a curated re-export. If `reasons/cluster.py` later adds or removes a public symbol, the shim's surface changes automatically.
3. **`noqa` annotation.** F401 (`imported but unused`) and F403 (`import *` used) are the two lint rules a shim triggers; silencing only those two keeps the rest of flake8 active.
4. **Docstring-as-migration-notice.** The one-line docstring names the preferred replacement import path — a convention for deprecation-style shims across the repo.

## Dependencies

**Imports (this file →):**
- `reasons.cluster` — the canonical implementation. The shim has a hard runtime dependency on it; if `reasons/cluster.py` is missing or fails to import, `import reasons_lib.cluster` fails too (there is no try/except, no fallback).

**Depended on by (`→ reasons_lib.cluster`):** per the tool output, `reasons/api.py`, `reasons/cli.py`, `reasons/cluster.py`, `reasons/contradictions.py`, `reasons/derive.py`, and the tests `test_api.py`, `test_cli.py`, `test_cluster.py`, `test_contradictions.py`, `test_derive.py`.

> ⚠️ A note for a maintainer: the "imported by" list is symmetric across the `reasons` and `reasons_lib` trees (e.g., `reasons/cluster.py` apparently "imports" the shim). This is almost certainly an artifact of the tooling treating the two trees as mirrors — a literal `from reasons_lib.cluster import *` inside `reasons/cluster.py` would be a circular import and would fail. Treat the list as "these modules share the same public API surface," not as a literal reverse-dependency graph.

## Flow

1. Someone runs `import reasons_lib.cluster`.
2. Python executes the shim module body:
   - Imports `reasons.cluster`. That module's top-level code runs (any constants, class definitions, function definitions, and any side effects it performs on import).
   - Performs the wildcard binding: for each name in `reasons.cluster`'s public namespace, assigns it into `reasons_lib.cluster.__dict__`.
3. The importing module can now call `reasons_lib.cluster.<symbol>` — the symbol is the *same object* as `reasons.cluster.<symbol>` (identity, not a copy).

There is no branching, no state, no lazy loading.

## Invariants

- **Single source of truth.** Any behavior in this file is behavior of `reasons.cluster`; editing the shim to add logic would break the "two paths are equivalent" invariant.
- **API parity.** `reasons_lib.cluster` and `reasons.cluster` expose the same public names at any moment after import. If a name is added to `reasons/cluster.py` (and not prefixed with `_`), it becomes importable from `reasons_lib.cluster` automatically.
- **No additional names.** The shim introduces nothing to `reasons_lib.cluster.__all__` (it doesn't even define one). It also won't leak names imported by *itself* (none).
- **Import order.** `reasons.cluster` must be importable before `reasons_lib.cluster` can be imported. This means the `reasons` package must be installed/available on `sys.path` in any environment that still imports `reasons_lib.cluster`.

## Error Handling

There is none — deliberately.

- If `reasons.cluster` raises at import time (e.g., a missing optional dependency, a config file it reads on load), `import reasons_lib.cluster` raises the same exception, with a traceback pointing into `reasons/cluster.py`. The shim does not wrap, catch, or re-raise with context.
- If a caller does `from reasons_lib.cluster import SomeName` and `SomeName` is not publicly defined in `reasons.cluster`, they get a normal `ImportError` — the shim has no fallback imports.
- No validation, no argument checking, no logging.

This matches the convention for a compatibility shim: keep the wrapper transparent so failures surface at the true origin (`reasons/cluster.py`).

## Topics to Explore

- [file] `reasons/cluster.py` — The real implementation; everything `reasons_lib/cluster.py` does is defined here.
- [file] `tests/test_cluster.py` — Exercises the cluster behavior; likely shows both import paths (`reasons.cluster` / `reasons_lib.cluster`) and the expected public API.
- [file] `reasons/api.py` — One of the top consumers of cluster functionality; shows how clustering is wired into the public API surface.
- [file] `reasons/derive.py` — Downstream of cluster; useful for seeing how clustering feeds derivation/inference logic.
- [general] `reasons-vs-reasons-lib-mirroring` — Confirm whether *every* file under `reasons_lib/` follows the same shim pattern and how the migration is tracked (CHANGELOG.md, deprecation policy).

## Beliefs

- `reasons-lib-cluster-is-a-shim` — `reasons_lib/cluster.py` contains no implementation of its own; it re-exports `reasons.cluster` via `from reasons.cluster import *`.
- `cluster-shim-requires-reasons-package` — Importing `reasons_lib.cluster` hard-fails with `ImportError` if `reasons.cluster` is not importable; there is no try/except fallback.
- `cluster-shim-preserves-name-identity` — Symbols accessed via `reasons_lib.cluster.X` are the same objects as `reasons.cluster.X`, not copies.
- `cluster-shim-inherits-api-from-reasons-cluster` — The set of names importable from `reasons_lib.cluster` is exactly `reasons.cluster`'s public (`__all__` or non-underscore) namespace; the shim performs no curation.
- `cluster-shim-suppresses-only-f401-and-f403` — The `# noqa: F401,F403` annotation silences only the "unused import" and "wildcard import" flake8 rules, leaving other linting of this file active.
