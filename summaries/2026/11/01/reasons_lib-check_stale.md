---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/check_stale.py

# `reasons_lib/check_stale.py` — Backwards-Compatibility Shim

## Purpose

This file is a **one-line re-export shim** that exists solely to keep old import paths working after the package was restructured from `reasons_lib` to `reasons`. Its entire responsibility is:

> Allow `from reasons_lib.check_stale import <anything>` to resolve identically to `from reasons.check_stale import <anything>`.

There is no logic here. No classes, no functions, no constants of its own. It is a transparent alias.

## Key Components

There are none. The file consists of:

| Line | Content | Role |
|------|---------|------|
| 1 | `"""Backwards-compatibility shim — use ``from reasons.check_stale`` instead."""` | Docstring telling consumers to migrate their imports |
| 2 | `from reasons.check_stale import *  # noqa: F401,F403` | The actual re-export |

The `noqa` comments suppress two Pyflakes warnings:
- **F401** – "imported but unused" (the star import is intentionally not bound locally)
- **F403** – "star imports are forbidden" (intentionally using `*` for forward-compatibility)

## Patterns

- **Star-import re-export**: A common Python idiom for deprecating a module path without pinning the exact symbol list. Any symbol that exists in `reasons.check_stale` (now or added later) is automatically available through this shim.
- **`reasons_lib/` as a deprecated namespace**: Looking at the repo tree, `reasons_lib/` mirrors most of `reasons/` but is missing several files (`mcp_client.py`, `mcp_server.py`, `hf.py`, `pubsub.py`, etc.), confirming it's a partial, legacy copy. The canonical implementation lives in `reasons/`.
- **Delegation over duplication**: Rather than copying the implementation, the shim delegates 100% to the canonical module, so there is a single source of truth.

## Dependencies

**Imports (outbound):**
- `reasons.check_stale` — the canonical module that actually contains the stale-reason-checking logic.

**Imported by (inbound):**
- `reasons/api.py`
- `reasons/pg.py`
- `reasons/verify.py`
- `tests/test_check_stale.py`
- `tests/test_check_stale_issue25.py`
- `tests/test_pin_sources.py`

> **Note:** It's interesting that modules *inside* `reasons/` (like `api.py`, `pg.py`, `verify.py`) still import through the `reasons_lib` path. This suggests the migration from `reasons_lib` → `reasons` is either incomplete or these internal imports haven't been updated yet. A future cleanup pass could switch them to `from reasons.check_stale import ...` directly, after which this shim could be deleted.

## Flow

There is no control flow. At import time:

1. Python executes `from reasons.check_stale import *`.
2. `reasons/check_stale.py` is loaded and all its public names (those without a leading `_`, or listed in `__all__`) are injected into the `reasons_lib.check_stale` namespace.
3. Any consumer that does `from reasons_lib.check_stale import check_stale_reasons` (or whatever symbols exist) gets the same objects as if they had imported from `reasons.check_stale` directly.

No transformation, no wrapping, no state.

## Invariants

- **Single source of truth**: All logic lives in `reasons/check_stale.py`. This file must never contain its own implementation.
- **Symbol completeness**: Because of `import *`, any new public symbol added to `reasons.check_stale` is automatically available here. No manual maintenance is needed (assuming `__all__` is respected or no symbols are underscore-prefixed).
- **No circular risk**: Since this file only imports from `reasons.check_stale` and contains no logic, it cannot introduce circular imports beyond what already exists in the canonical module.

## Error Handling

There is none. This file:

- Cannot raise exceptions at call time (there are no callables).
- Will raise `ImportError` / `ModuleNotFoundError` **at import time** if `reasons.check_stale` itself fails to import (e.g., a missing transitive dependency).
- Does not catch, suppress, or transform any errors.

In short: if the canonical module is healthy, this shim is invisible; if it's broken, the failure surfaces as a standard import error pointing at `reasons.check_stale`.

---

## Topics to Explore

- [file] `reasons/check_stale.py` — The canonical implementation this shim delegates to; understanding what "stale" means in the context of reasons/beliefs is essential.
- [repo] `reasons_lib/` vs `reasons/` — The full scope of the rename; which modules still lack a shim and which `reasons/` internal imports still reference the old namespace.
- [file] `tests/test_check_stale_issue25.py` — A regression test tied to a specific issue; reveals edge cases and invariants the stale-check logic must uphold.
- [general] `deprecation-strategy` — How the project handles breaking import-path changes across 30+ mirrored modules; whether a removal timeline or CI guard exists for `reasons_lib/`.
- [function] `reasons/check_stale.py:check_stale_reasons` — The actual entry point (assumed name) that determines which beliefs have gone stale; the core business logic behind this shim.

---

## Beliefs

- `reasons_lib-is-a-reasons` — Every file in `reasons_lib/` is a star-import shim that re-exports from the same-named module in `reasons/`; no logic is duplicated.
- `shim-maps-to-canonical-namespace` — `reasons_lib.check_stale` and `reasons.check_stale` expose identical public symbols at any given point in time.
- `reasons-internal-imports-stale` — At least three modules inside `reasons/` (`api.py`, `pg.py`, `verify.py`) still import through the deprecated `reasons_lib` path rather than their own package.
- `shim-produces-no-side-effects` — Importing `reasons_lib.check_stale` executes no code beyond the import of `reasons.check_stale`; it has no module-level state, no logging, no configuration reads.
- `reasons-contains-reasons_lib` — The `reasons_lib/` directory is a strict subset of `reasons/` (missing `mcp_client.py`, `mcp_server.py`, `hf.py`, `pubsub.py`, and several others), meaning not all canonical modules have a legacy shim.
