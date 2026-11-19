---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons_lib/repair.py

# `reasons_lib/repair.py`

```python
"""Backwards-compatibility shim — use ``from reasons.repair`` instead."""
from reasons.repair import *  # noqa: F401,F403
```

## Purpose

This module is a **re-export shim** (a.k.a. compatibility alias / facade). The actual `repair` implementation lives in `reasons/repair.py`. `reasons_lib/repair.py` exists only to keep legacy imports like `from reasons_lib.repair import …` or `import reasons_lib.repair as repair` working after the codebase migrated to the `reasons` package name.

It owns **no behavior, state, or logic of its own**. Its sole responsibility is to make the public API of `reasons.repair` available under the `reasons_lib.repair` module path.

The docstring is the authoritative statement of intent: *"Backwards-compatibility shim — use `from reasons.repair` instead."* That is a soft-deprecation notice — new code should import from the canonical package.

## Key Components

There are **none defined locally** — no classes, functions, constants, or `__all__`. The module's namespace is populated entirely by Python's wildcard-import machinery.

- `from reasons.repair import *` — this is the entire file. If `reasons/repair.py` defines `__all__`, only those names are re-exported. If not, Python re-exports every name that does not start with `_` (dunders and private helpers are excluded, unless `__all__` explicitly lists them).
- `# noqa: F401,F403` — silences the linter:
  - **F401** (imported-but-unused) fires because the star import has no local references; the names it brings in are meant to be re-exported, not used here.
  - **F403** (unable to detect undefined names) fires because `import *` defeats static analysis of this module's namespace.

## Patterns

- **Shim / compatibility alias.** Classic single-line module whose entire body is a re-export. Paired with the package-level directory listing that shows `reasons_lib/` mirrors `reasons/` file-for-file (see also `reasons_lib/repair_premises.py`, `reasons_lib/api.py`, etc.).
- **Wildcard re-export (`from X import *`).** Chosen because it lets new symbols added to `reasons.repair` automatically surface through the shim **without editing this file** — the shim stays a one-liner regardless of how large `reasons.repair` grows.
- **Soft deprecation via docstring.** No `DeprecationWarning` is emitted, so this is a "please migrate" note rather than an enforced migration; callers won't see runtime warnings today.

## Dependencies

**Imports (depends on):**
- `reasons.repair` — the real implementation. This shim is entirely parasitic on it.

**Imported by (consumers):**
- `reasons/api.py` — notable: the **canonical** package imports through the **legacy** shim, not the reverse. This creates a visible cross-package edge `reasons.api → reasons_lib.repair → reasons.repair`. Because the shim only re-binds names, no circular-import hazard is introduced, but it does mean the "legacy" package is still on the live import path, not just kept alive for old tests.
- `reasons_lib/repair_premises.py` — an intra-`reasons_lib` consumer.
- `tests/test_repair.py`, `tests/test_repair_premises.py`, `tests/test_research.py` — tests still exercise the `reasons_lib` path.

## Flow

There are no function calls, branches, or data transformations — only module initialization:

1. A caller triggers `import reasons_lib.repair` (directly, or transitively via `reasons.api` or `reasons_lib.repair_premises`).
2. Python executes this file top-to-bottom.
3. The single statement resolves `reasons.repair`, executes it if not already in `sys.modules`, then copies each public name from that module into the local `reasons_lib.repair` namespace.
4. The module object is registered in `sys.modules['reasons_lib.repair']` with the merged namespace.

Because the bound objects are **the same object identities** as in `reasons.repair`, any call through `reasons_lib.repair.f(...)` is indistinguishable at runtime from `reasons.repair.f(...)`. Their `__module__` attribute still reads `"reasons.repair"`, which matters for `repr`, pickling, and introspection.

## Invariants

- **Namespace parity is delegated to `reasons.repair`.** Anything new added to `reasons.repair` appears here automatically on a fresh import, but only if it is public (no leading underscore) or listed in `reasons.repair.__all__`.
- **`__all__` is not itself re-exported.** The star import copies the *members* named in `reasons.repair.__all__`, but not the `__all__` attribute itself (dunders are excluded unless listed). So `reasons_lib.repair` typically has no `__all__` of its own.
- **No private-name leakage.** Names starting with `_` in `reasons.repair` are not reachable as `reasons_lib.repair._foo` unless `reasons.repair.__all__` explicitly includes them.
- **No state divergence at import time.** Every attribute bound here is the identical object as in the source module; there is no copy or wrapper.
- **Runtime mutation is one-directional.** If code dynamically installs a new attribute on `reasons.repair` after this shim has been imported, that attribute will *not* automatically appear on `reasons_lib.repair` (they are separate module `__dict__`s). Conversely, attributes assigned directly to `reasons_lib.repair` shadow only that module — they do not modify `reasons.repair`.
- **Import order is enforced by Python.** `reasons.repair` must be importable; there is no lazy proxy, so any import error there surfaces here.

## Error Handling

**None of its own.** The shim:

- Does **not** wrap the import in `try/except` — an `ImportError` (e.g., a missing optional dependency inside `reasons/repair.py`) propagates unchanged, and the traceback will point at `reasons/repair.py`, not this file.
- Does **not** emit a `DeprecationWarning` — the docstring is the only signal that the path is legacy.
- Does **not** validate that expected names exist — if `reasons.repair` renames or removes a symbol, `reasons_lib.repair` silently loses it, and the failure will surface at the caller's import site, not here.

Net effect: error semantics are *exactly* those of `reasons.repair`; this file adds no new failure modes except being imported before its upstream is importable.

---

## Topics to Explore

- [file] `reasons/repair.py` — The real implementation this shim re-exports; understand its public surface (`__all__` or lack thereof) to know exactly what crosses the boundary.
- [file] `reasons/api.py` — Why the canonical package imports through the legacy shim; check whether this is a lingering dependency that could be redirected to `reasons.repair` directly.
- [general] `reasons-lib-to-reasons-migration` — The dual-package layout (`reasons/` and `reasons_lib/` mirrored 1:1) and the migration strategy that produced these shims; useful for knowing when a file should be deleted vs. kept.
- [file] `reasons_lib/__init__.py` — Whether the shim package does anything at package level (namespace, warnings, lazy loading) beyond per-module shims.
- [function] `reasons_lib/repair_premises.py` — A live intra-shim consumer; check whether it imports the shim or the real module and whether that pattern is consistent across `reasons_lib/`.

## Beliefs

- `reasons-lib-repair-re-exports-reasons-repair` — This module defines no symbols locally; its entire namespace is produced by `from reasons.repair import *`.
- `reasons-lib-repair-requires-reasons-repair` — Importing `reasons_lib.repair` transitively imports `reasons.repair`; an import error there propagates with no fallback or wrapping.
- `reasons-api-imports-reasons-lib-repair` — The canonical `reasons/api.py` consumes the legacy shim rather than importing `reasons.repair` directly.
- `reasons-lib-repair-is-a-compat-shim` — The docstring identifies the module as a backwards-compatibility shim and directs new code to `reasons.repair`.
- `reasons-lib-repair-preserves-underscore-names-only-via-all` — Names in `reasons.repair` beginning with `_` are re-exported here only if `reasons.repair.__all__` lists them; otherwise they are dropped by the wildcard import.
