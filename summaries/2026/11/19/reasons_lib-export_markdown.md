---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/export_markdown.py

# `reasons_lib/export_markdown.py`

```python
"""Backwards-compatibility shim — use ``from reasons.export_markdown`` instead."""
from reasons.export_markdown import *  # noqa: F401,F403
```

That's the whole file: a two-line re-export. Everything real lives in `reasons/export_markdown.py`. The repo layout confirms the intent — `reasons_lib/` and `reasons/` contain an almost identical module list (`api.py`, `ask.py`, …, `export_markdown.py`, …), so `reasons_lib` is the **legacy package name** and `reasons` is the **canonical** one.

---

## 1. Purpose

- **Owns nothing.** No functions, no classes, no constants, no state. Its sole responsibility is name forwarding: any symbol that used to be importable as `reasons_lib.export_markdown.X` remains importable under that path after the code was relocated to `reasons.export_markdown`.
- **Preserves the public import path** for downstream users and for in-repo callers that were never migrated (`reasons/api.py` still imports it, as do five test modules).
- **Signals deprecation** to humans via the module docstring; there is no `DeprecationWarning` emitted (see §7).

## 2. Key Components

| Component | Contract |
|---|---|
| Module docstring (line 1) | Declares the canonical path `reasons.export_markdown`. Purely advisory — not machine-enforced. |
| `from reasons.export_markdown import *` (line 2) | At import time, binds every name the target module exports into this module's globals. |
| `# noqa: F401,F403` | Suppresses flake8 *unused-import* (F401) and *star-import-used* (F403). Makes intent explicit: the imports are deliberate re-exports. |

The **effective public surface** of this file is not defined here; it is inherited:

- If `reasons/export_markdown.py` defines `__all__`, that list is exactly the re-exported set.
- If it does **not**, Python's star-import rule applies: every global in the target whose name does not begin with `_` is copied — **including the target's own imports** (`Path`, `json`, dataclasses, storage helpers, etc.). That is the usual way shims accidentally leak internals.

There is **no `__all__` in this file**, so `from reasons_lib.export_markdown import *` delegates to the same rule transitively, and `dir(reasons_lib.export_markdown)` reflects whatever landed in the namespace plus the ambient module attributes.

## 3. Patterns

- **Facade / re-export shim (alias module).** Canonical implementation + thin compatibility module. Deliberately contains zero logic so there is never a second source of truth for markdown-export behavior.
- **Package rename with a deprecation runway.** `reasons_lib` → `reasons` is a breaking rename; per-module shims give a migration window where both paths work.
- **Wildcard re-export** rather than an explicit `from reasons.export_markdown import render_markdown, export_notes, …`. Cheaper to maintain (nothing to update when the target gains a symbol) but couples the shim's surface to the target's `__all__` discipline.
- **Lint suppression as documentation.** `noqa` codes state "this import exists for its side effect of populating the namespace."

## 4. Dependencies

**Depends on:**
- `reasons.export_markdown` — the only runtime dependency, resolved at import time.
- Transitively everything that module imports (storage, metadata, etc.) — importing the shim therefore pays the full import cost of the real exporter.

**Depends on it (`imported_by`):**
- `reasons/api.py` — note the direction: the *canonical* package still reaches the exporter through the *legacy* path. Likely a leftover from the rename; a good cleanup candidate.
- `tests/test_export_markdown.py`, `tests/test_metadata.py`, `tests/test_metadata_extended.py`, `tests/test_roundtrip.py`, `tests/test_source_type.py` — these exercise the exporter, and several of them load it through `reasons_lib`, which means they double as regression tests for the shim.

No third-party imports, no I/O, no module-level side effects beyond the star import.

## 5. Flow

Import-time only; there is no callable code to trace.

1. `import reasons_lib.export_markdown` → the module body runs.
2. Python imports `reasons.export_markdown` (executing its entire module body once; cached in `sys.modules` thereafter).
3. The star-import copies the export set into `reasons_lib.export_markdown.__dict__`.
4. Done. Any later `reasons_lib.export_markdown.foo(...)` invokes the *same function object* as `reasons.export_markdown.foo(...)` — identity, `isinstance`, and `pickle` by qualified name all resolve to the canonical module.

Runtime data transformation: none. The shim is a pure aliasing layer.

## 6. Invariants

- **Single source of truth.** No logic may be added here. If behavior needs changing, it changes in `reasons/export_markdown.py`. Anything implemented locally would be invisible to canonical-path consumers and produce split-brain behavior.
- **Surface ⊆ target surface.** A name exists at this path iff `reasons.export_markdown` exports it. Deleting a public symbol from the target silently removes it from the legacy path too — the shim cannot preserve names the target dropped.
- **Import-time atomicity.** Either the target imports cleanly and all exports land, or the whole shim fails to import (see §7). There is no partial state.
- **Shared identity, separate namespaces.** `reasons_lib.export_markdown` and `reasons.export_markdown` are *distinct module objects* with *shared object references*. Re-binding a name on the shim (`reasons_lib.export_markdown.render = fake`) affects only the shim's global; the canonical module is untouched, and call sites that captured the original reference are unaffected. This is the single most important subtlety for test authors here.
- **No cycle created.** `reasons/export_markdown.py` must not import `reasons.api` (which imports this shim); with a star import that cycle would surface as a partially-initialized module.

## 7. Error Handling

There is deliberately **no** error handling — nothing is caught, wrapped, or downgraded.

- **Missing/broken target:** if `reasons.export_markdown` fails (bad syntax, missing dependency, its own `ImportError`), that exception propagates out of *this* module's import. Python reports it against `reasons_lib/export_markdown.py` line 2, which can be misleading in tracebacks because the real failure is one package over.
- **Missing symbol:** `from reasons_lib.export_markdown import render_markdown` raises `AttributeError`/`ImportError` if the target does not export it. Because the shim has no `__all__` of its own, there is no early validation that the expected names exist — a rename in the target is caught only when a downstream importer breaks.
- **Silent under-export:** if the target defines `__all__` and forgets a symbol that used to be public, the shim degrades silently; legacy callers get an `ImportError` with no deprecation hint.
- **No `DeprecationWarning`.** Unlike a typical shim, importing this path emits no warning, so clients receive no runtime nudge to migrate. Migration is enforced only by documentation and eventual deletion.

---

## Topics to Explore

- [file] `reasons/export_markdown.py` — The canonical implementation this shim forwards to; check whether it defines `__all__` (which determines whether internal imports leak through the shim) and what the public export set actually is.
- [file] `reasons/api.py` — The canonical package importing through the legacy `reasons_lib` path; confirms the rename is incomplete and identifies a cleanup target.
- [file] `tests/test_export_markdown.py` — Shows which import path the tests use and whether they monkeypatch symbols through the shim (a pattern that silently fails on the canonical module).
- [general] `reasons-lib-to-reasons-migration` — Are the remaining `reasons_lib/*.py` files all shims like this one, or do some still hold real logic? Determines how far the rename has progressed and what is safe to delete.
- [file] `pyproject.toml` — Whether both `reasons` and `reasons_lib` are shipped in the package/discovery config; decides if this shim is still reachable by external installers.

## Beliefs

- `reasons-lib-export-markdown-is-a-shim` — The file contains no implementation; it re-exports every public name from `reasons.export_markdown`.
- `export-markdown-shim-requires-reasons-export-markdown` — Importing `reasons_lib.export_markdown` raises immediately if `reasons.export_markdown` cannot be imported; failures are reported at line 2 of the shim.
- `export-markdown-shim-has-no-__all__` — The shim defines no `__all__`, so its exported surface is inherited verbatim from the target's `__all__` or, absent that, from the target's non-underscore globals (including its imports).
- `reasons-api-depends-on-reasons-lib-export-markdown` — The canonical `reasons` package still imports the exporter via the legacy `reasons_lib` path.
- `rebinding-shim-attributes-does-not-affect-canonical-module` — `reasons_lib.export_markdown` and `reasons.export_markdown` are distinct module objects sharing references; assigning to an attribute on the shim leaves the canonical module unchanged.
- `export-markdown-shim-emits-no-deprecation-warning` — Neither the import nor any access emits a warning; migration is enforced only by the docstring.
