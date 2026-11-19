---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/derive.py

# `reasons_lib/derive.py` — Backwards-Compatibility Shim

```python
"""Backwards-compatibility shim — use ``from reasons.derive`` instead."""
from reasons.derive import *  # noqa: F401,F403
```

That is the entire file. Its docstring is the contract; the rest is a one-line forwarding import. Everything below is what a maintainer needs to know about *why a two-line file matters*.

---

## 1. Purpose

The repository ships two parallel packages with identical module trees (`reasons/…` and `reasons_lib/…`). `reasons_lib/derive.py` is the **legacy alias** for `reasons/derive.py` — the real implementation lives under the `reasons` package (the rename direction is stated explicitly in the docstring: *"use `from reasons.derive` instead"*).

This file owns **zero logic**. Its sole responsibility is namespace forwarding: any code that still does

```python
from reasons_lib.derive import <something>
import reasons_lib.derive
```

continues to work after the package rename, resolving `<something>` to the object defined in `reasons.derive`.

It exists so that the rename can be staged — new code targets `reasons.*`, old code (and possibly external consumers, published import paths, notebooks, or `pyproject.toml` entry points) keeps working unmodified.

---

## 2. Key Components

There are **no classes, functions, or constants defined here**. The "components" are the module-level mechanics:

| Component | Contract |
|---|---|
| Module docstring | Declares this module deprecated in favor of `reasons.derive`. This is the only human-readable signal a maintainer gets at import time. |
| `from reasons.derive import *` | Binds every **public** name of `reasons.derive` (i.e. every name in that module's `__all__`, or — if `__all__` is undefined — every name not starting with `_`) into `reasons_lib.derive.__dict__`. |
| `# noqa: F401,F403` | Suppresses flake8/ruff **F401** (imported but unused) and **F403** (star import). Without it, linting/CI would flag this file. |

### What it re-exports, concretely

Because the re-export is a **star import**, the exported surface is delegated entirely to `reasons/derive.py`'s `__all__` (or its namespace). Tests `tests/test_derive.py`, `tests/test_derive_budget.py`, and `tests/test_source_type.py` import from this module, which confirms the public derivation API — whatever `derive(...)`, budget handling, and source-type classification symbols are — must be reachable through `reasons.derive`'s public surface for those tests to pass.

**Not re-exported:**
- Any name starting with `_` (private helpers) — unless `__all__` explicitly lists it.
- Submodules of `reasons` (a star import of a module does not pull in sibling packages).
- The name `__all__` itself — `reasons_lib.derive` does **not** define `__all__`, so `from reasons_lib.derive import *` would fall back to "all non-underscore names," which will also expose incidental imports that leaked into the shim's namespace.

---

## 3. Patterns

- **Shim / deprecation-alias module** — the classic `old_pkg/mod.py` → `from new_pkg.mod import *` pattern used during package renames. It's a degenerate Adapter (no behavior translation, identity mapping only).
- **Re-export via star import** — chosen over explicit `from reasons.derive import foo, bar, baz` so the shim never has to be touched when the real API grows. Cost: the shim's public surface is **implicit** and invisible to static analysis tooling without following the target's `__all__`.
- **Lint suppression with reason codes** — `# noqa: F401,F403` names the exact codes rather than a blanket `# noqa`, so new unrelated warnings are not silenced.
- **Identity preservation** — the re-exported objects are the *same* objects, not copies or wrappers. `reasons_lib.derive.SomeClass is reasons.derive.SomeClass` ⇒ `isinstance` checks, `pickle` round-trips, and exception classes all behave identically regardless of which import path produced them.

---

## 4. Dependencies

**Depends on (imports):**
- `reasons.derive` — hard runtime dependency. Importing the shim *imports the real module*, executing its module-level code (registration, `__all__` construction, any module-scope setup).

**Depended on by (`imported_by`):**
- `reasons/api.py` — the **new** package still imports through the **legacy** shim. This is the most important line in this analysis: the rename is only half-complete internally.
- `reasons/cli.py` — same; the CLI path also goes through the shim.
- `tests/test_derive.py`, `tests/test_derive_budget.py`, `tests/test_source_type.py` — test suites pinned to the legacy import path.

Since `api.py` and `cli.py` are effectively the program's front doors, **this shim is on the hot path**, not dead code. Removing it today would break both the CLI and the public API module.

---

## 5. Flow

```
caller: import reasons_lib.derive
        │
        ├─ Python imports reasons_lib package (reasons_lib/__init__.py)
        │
        ├─ executes reasons_lib/derive.py body
        │     └─ `from reasons.derive import *`
        │           ├─ import reasons  → reasons/__init__.py
        │           ├─ import reasons.derive → executes real module top-to-bottom
        │           │     └─ builds its namespace / __all__
        │           └─ copy public names into reasons_lib.derive.__dict__
        │
        └─ returns module object `reasons_lib.derive`
```

Consequences of this flow:

1. **Two module objects exist in `sys.modules`**: `reasons.derive` and `reasons_lib.derive`. Any module-level mutable state (caches, registries, counters) lives in `reasons.derive`. The shim is a *snapshot of name bindings*, not a live view.
2. **Import side effects run exactly once** — misdirected to `reasons.derive`. If `reasons/derive.py` registers anything (e.g. a CLI subcommand, a metadata handler), it's registered once, not twice.
3. **No data transformation, no branching, no I/O.** Control flow terminates at the import completion.

---

## 6. Invariants

- **`reasons.derive` must be importable without importing `reasons_lib.derive`.** If `reasons/derive.py` (or anything it transitively imports) ever imports back from `reasons_lib.derive`, you get a **partially-initialized module / circular import**, and the star import will silently bind a truncated set of names (or raise `ImportError`/`AttributeError`). The dependency direction is strictly one-way: `reasons_lib → reasons`.
- **The public API of `reasons.derive` is the single source of truth.** There is no duplication; adding/removing a symbol in `reasons/derive.py` immediately changes what this shim exports. Never edit this file to add functionality.
- **Underscore names are not forwarded.** A consumer relying on `reasons_lib.derive._private_helper` will break even though it "worked" before the rename.
- **Object identity is preserved.** `reasons_lib.derive.X is reasons.derive.X` for every re-exported `X`.
- **Rebinding through the shim is local.** `reasons_lib.derive.foo = replacement` (or `monkeypatch.setattr("reasons_lib.derive.foo", …)`) only changes the shim's binding. Functions inside `reasons.derive` that call the *global* `foo` in their own module namespace are **unaffected**. This is a real monkeypatching trap given `reasons/api.py` and `reasons/cli.py` route through the shim — a test that patches the legacy path may pass while the production path (direct `reasons.derive` import) sees the original.
- **No `__all__` in the shim** ⇒ `from reasons_lib.derive import *` leaks any incidental module-level names that survived into `__dict__` (dunders excepted). Don't rely on it for a curated surface.

---

## 7. Error Handling

There is **no error handling** — and that is correct for a shim:

- `ImportError` / `ModuleNotFoundError` from `reasons.derive` **propagates unchanged**. If the real module is missing or fails to import, the caller sees the underlying failure, not a masked "legacy module unavailable" error. This is desirable: the traceback points at the real cause.
- No `try/except ImportError: pass` fallback — a common shim anti-pattern that would silently produce a half-empty module. Its absence means a broken `reasons.derive` breaks the legacy path loudly, which is the right failure mode during a rename.
- No validation, no deprecation warning emission (`warnings.warn(..., DeprecationWarning)` is *not* used). If you want runtime deprecation telemetry for legacy importers, you'd have to add a module-level `warnings.warn` here — currently the deprecation is only documented in the docstring.
- No defensive `__getattr__` proxy, so attribute lookups that miss the star import raise a plain `AttributeError` from the shim module rather than being lazily forwarded.

---

## Topics to Explore

- [file] `reasons/derive.py` — The real implementation; this shim's entire public surface is defined by its `__all__` (or lack thereof). Read it first before touching this alias.
- [file] `reasons/api.py` — Still imports through the legacy shim despite being in the new package; a candidate cleanup that would let `reasons_lib` be dropped.
- [file] `tests/test_derive_budget.py` — Exercises the derivation budget logic via the legacy import path; useful to see which symbols the shim must keep alive.
- [file] `tests/test_source_type.py` — Another legacy-path consumer; shows source-type classification is part of the re-exported API.
- [general] `package-rename-shim-strategy` — How `reasons_lib` → `reasons` was staged, whether any external import paths are still supported, and when/whether the shim can be deleted.

## Beliefs

- `reasons-lib-derive-is-a-shim` — `reasons_lib/derive.py` contains no logic beyond a docstring and a star import of `reasons.derive`.
- `reasons-lib-derive-reexports-reasons-derive` — All public symbols of `reasons.derive` are bound into `reasons_lib.derive` with identical object identity.
- `reasons-lib-derive-requires-reasons-derive-importable` — Import failures in `reasons.derive` propagate unchanged through the shim; there is no fallback or `try/except`.
- `reasons-api-depends-on-legacy-shim` — `reasons/api.py`, `reasons/cli.py`, and three test modules import `reasons_lib.derive` rather than `reasons.derive` directly.
- `reasons-lib-derive-excludes-private-names` — Because the shim uses `import *` with no `__all__` override, underscore-prefixed names in `reasons.derive` are not forwarded.
- `reasons-lib-derive-defines-no-all` — The shim does not define `__all__`, so a star import of the shim exposes every non-underscore binding in its namespace.
