---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/network.py

# `reasons_lib/network.py`

## Purpose

This file is a **backwards-compatibility shim**. Its sole job is to re-export the entire public API of `reasons.network` into the `reasons_lib.network` namespace. It exists so that existing consumers who still do `from reasons_lib.network import Network, ...` continue to work after the project's canonical package name changed from `reasons_lib` to `reasons`.

It owns no logic. It has no state. It is a transparent alias.

## Key Components

There are no classes, functions, or constants defined here. The entire module body is:

```python
from reasons.network import *  # noqa: F401,F403
```

- **`from reasons.network import *`** — Re-exports every name in `reasons.network`'s `__all__` (or, if `__all__` is undefined, every public name not starting with `_`). This means `Network`, `Justification`, `Defeater`, and any other public symbols from the real module become accessible via `reasons_lib.network`.
- **`# noqa: F401,F403`** — Suppresses two `flake8`/`ruff` diagnostics:
  - `F401` — "imported but unused" (the star import is deliberately unused in this file's local scope; it exists to populate `reasons_lib.network`'s namespace for *other* importers).
  - `F403` — "star- imports not allowed" (this is an intentional, well-understood star import for a compatibility shim).

## Patterns

- **Namespace alias / shim module**: The project migrated its import path from `reasons_lib.*` → `reasons.*`. Rather than breaking every downstream consumer, each file in the old `reasons_lib/` package is a one-line re-export. This is the standard Python "deprecation shim" pattern (analogous to `typing.io` → `io`, or `collections.abc` aliases in 3.3).
- **Mirror directory structure**: The repo tree shows `reasons/` and `reasons_lib/` are near-identical file-by-file. Every file in `reasons_lib/` follows this same shim pattern. The canonical code lives in `reasons/`.
- **Flat re-export (no selective filtering)**: Because the shim uses `*`, it automatically tracks any new public symbols added to `reasons.network` in the future without requiring the shim to be updated. This is both a convenience and a risk (see Invariants).

## Dependencies

| Direction | Module | Relationship |
|-----------|--------|--------------|
| **Depends on** | `reasons.network` | The single, hard dependency. If `reasons.network` is removed or renamed, this shim breaks at import time. |
| **Depended on by** | `reasons/api.py`, `reasons/check_stale.py`, `reasons/compact.py`, `reasons/export_card.py`, `reasons/export_markdown.py`, `reasons/import_agent.py`, `reasons/import_beliefs.py`, `reasons/storage.py` | Internal modules that still use the old import path (likely a transitional state; these should eventually migrate to `reasons.network`). |
| **Depended on by** | 25+ test files in `tests/` | The test suite still imports via `reasons_lib.network`, meaning tests exercise the shim path rather than the canonical one. |

> **Note:** The "imported by" list includes files inside `reasons/` itself. This is worth flagging — the canonical package is importing from its own compat shim. This works (Python resolves it) but adds an unnecessary indirection layer.

## Flow

Execution is trivial and happens **once at import time**:

1. Python's import machinery encounters `reasons_lib/network.py`.
2. It executes the single `from reasons.network import *` statement.
3. Python imports `reasons.network` (running its full module body — class definitions, function definitions, etc.).
4. The star import copies all public names from `reasons.network.__dict__` into `reasons_lib.network.__dict__`.
5. Any subsequent `from reasons_lib.network import Network` succeeds, resolving to the same object defined in `reasons.network`.

There is no runtime control flow, no data transformation, no method dispatch. The shim is inert after import.

## Invariants

- **Single source of truth**: All logic lives in `reasons/network.py`. The shim in `reasons_lib/network.py` must never contain independent definitions. If a developer adds a class here, it will silently shadow the canonical one for anyone importing via `reasons_lib`.
- **Name parity**: The shim exposes exactly the public names of `reasons.network`. If a symbol is made private (prefixed with `_`) in `reasons.network`, it automatically disappears from the shim — consumers relying on it will get an `ImportError`.
- **No circular imports**: `reasons.network` must not (transitively) import from `reasons_lib.*`, or you get a circular import error at module load time.

## Error Handling

There is **none** in this file. The shim does not catch, wrap, or log anything.

- If `reasons.network` fails to import (e.g., a syntax error, a missing dependency like `pydantic`), the `ImportError` propagates directly to the caller of `from reasons_lib.network import ...`.
- If a specific name is requested that no longer exists in `reasons.network` (e.g., `from reasons_lib.network import OldClass`), a standard `ImportError: cannot import name 'OldClass' from 'reasons_lib.network'` is raised by the import machinery.
- There is no deprecation warning (`DeprecationWarning`) emitted. Consumers get no signal that they're on the old path. This is a deliberate (or accidental) omission — a `warnings.warn(..., DeprecationWarning)` call would be a natural addition if the project wanted to nudge users toward `reasons.network`.

---

## Topics to Explore

- [file] `reasons/network.py` — The canonical implementation this shim re-exports; understanding `Network`, `Justification`, and `Defeater` is prerequisite to understanding the domain model.
- [file] `reasons/__init__.py` vs `reasons_lib/__init__.py` — Compare the two package inits to see whether `reasons_lib` declares itself deprecated or registers any import hooks.
- [repo] `reasons_lib/` — Audit all 30+ shim files to confirm they are all single-line re-exports and that none have drifted (i.e., none contain divergent logic).
- [file] `reasons/storage.py` — It imports from `reasons_lib.network` *within* the canonical package; trace why this internal module hasn't been migrated to `reasons.network` yet.
- [general] `deprecation-timeline` — Check `CHANGELOG.md` and `pyproject.toml` for a planned removal date or version gate for the `reasons_lib` compatibility layer.

---

## Beliefs

- `reasons-lib-network-is-a-reasons-network-alias` — `reasons_lib.network` exposes exactly the same public names and objects as `reasons.network`; there is no additional logic, state, or behavior in the shim.
- `reasons-internal-modules-still-import-via-reasons-lib` — At least 8 modules inside the canonical `reasons/` package (e.g., `reasons/api.py`, `reasons/storage.py`) import from `reasons_lib.network` rather than `reasons.network`, introducing an unnecessary indirection that will break if the shim is removed.
- `shim-star-import-tracks-reasons-network-__all__` — The `import *` in the shim means any new public symbol added to `reasons.network` is automatically visible through `reasons_lib.network` without modifying the shim, but any symbol made private (`_`-prefixed) is silently dropped.
- `shim-emits-no-deprecation-warning` — Importing from `reasons_lib.network` produces no `DeprecationWarning` or `PendingDeprecationWarning`, so consumers receive no programmatic signal to migrate to `reasons.network`.
- `test-suite-exercises-shim-path-not-canonical` — 25+ test files import from `reasons_lib.network`, meaning the test suite validates the shim's import chain but does not independently validate that `reasons.network`'s symbols are correctly exported if the shim were removed.
