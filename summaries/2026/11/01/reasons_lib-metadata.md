---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/metadata.py

# `reasons_lib/metadata.py`

## Purpose

This file is a **backwards-compatibility shim**. Its sole responsibility is to ensure that any code still importing from `reasons_lib.metadata` continues to work, even though the canonical implementation has moved to `reasons.metadata`. The docstring is explicit: *"use ``from reasons.metadata`` instead."*

It owns no logic, no state, and no behavior. It is a thin indirection layer that will (presumably) be deleted once all consumers are migrated to the `reasons.*` namespace.

## Key Components

There is exactly one statement:

```python
from reasons.metadata import *  # noqa: F401,F403
```

- **`import *`** — re-exports every public name (anything not starting with `_`) defined in `reasons.metadata`. This means the full public surface of the metadata module — dataclasses, constants, helper functions — is transparently available under `reasons_lib.metadata`.
- **`# noqa: F401,F403`** — suppresses two flake8/pyflakes warnings:
  - `F401`: "imported but unused" (expected, since we're re-exporting)
  - `F403`: "'from module import *' used" (expected, by design)

There are no classes, functions, or constants defined *in this file*.

## Patterns

- **Re-export / facade pattern**: This is the classic "thin wrapper" used during package renames or splits. The `reasons_lib/` package mirrors `reasons/` module-for-module, and every file in `reasons_lib/` is likely a one-liner star-import from its `reasons/` counterpart.
- **Deprecation-by-shim**: Rather than removing the old import path (which would break downstream consumers in one release), the project keeps a working alias. The docstring nudges new code toward the canonical path.
- **Dual-package layout**: `reasons/` is the active implementation; `reasons_lib/` is the legacy alias. This mirrors the old Python "src-layout vs flat-layout" migration pattern, or the transition from `package_v1` → `package_v2`.

## Dependencies

**Depends on (imports):**
- `reasons.metadata` — the actual implementation. All names come from there.

**Depended on (imported by):**
- `reasons/api.py`, `reasons/cli.py`, `reasons/export_card.py`, `reasons/export_markdown.py`, `reasons/metadata.py`, `reasons/pg.py`, `reasons/storage.py` — internal modules that still use the old import path
- `tests/test_export_card.py`, `tests/test_metadata.py`, `tests/test_metadata_extended.py` — test suite

The breadth of the "imported by" list (9 consumers) tells you the migration to `reasons.metadata` is **incomplete**. Those 9 files still resolve through this shim.

## Flow

Execution is trivial:

1. Python's import system encounters `import reasons_lib.metadata` (or `from reasons_lib.metadata import X`).
2. The module body runs: the star import pulls all public names from `reasons.metadata` into `reasons_lib.metadata.__dict__`.
3. The importing module sees those names as if they were defined locally.
4. No code executes at import time beyond what `reasons.metadata` itself runs (e.g., dataclass definitions, constant assignments).

There is no conditional logic, no lazy loading, no `__getattr__` trickery — it's a static, eager re-export.

## Invariants

- **Name parity**: Every public name in `reasons.metadata` is guaranteed to be available in `reasons_lib.metadata`. If a name is added or removed in `reasons.metadata`, the shim picks it up automatically (star import).
- **No independent state**: This module cannot diverge from `reasons.metadata`. There is no way to override, filter, or extend the re-export here.
- **Forward-only**: You cannot import something *from* this shim that doesn't exist in `reasons.metadata`. It is strictly a subset (or exact set) of the canonical module's public API.

## Error Handling

There is no custom error handling. Two failure modes are possible:

1. **Import-time failure**: If `reasons.metadata` itself fails to import (missing dependency, syntax error), the exception propagates up through this module's import. The `from reasons_lib.metadata import *` line is the trigger point in the traceback.
2. **AttributeError at use-time**: If a consumer does `from reasons_lib.metadata import SomeNewName` but `SomeNewName` was never added to `reasons.metadata`, the error surfaces as a standard `ImportError`/`AttributeError` from the star import resolution. There is no custom message or graceful degradation.

Errors are never swallowed. The `# noqa` comments suppress *static* analysis warnings, not runtime errors.

---

## Topics to Explore

- [file] `reasons/metadata.py` — The actual implementation this shim re-exports; understand the dataclasses, constants, and helper functions that constitute the metadata API
- [file] `reasons_lib/__init__.py` — Check whether the shim pattern is consistent across all `reasons_lib/` modules or if some have additional logic
- [general] `reasons-lib-migration-status` — How many of the 9 importers still use `reasons_lib.*` vs `reasons.*`; a grep for `from reasons_lib` will reveal the remaining migration debt
- [file] `tests/test_metadata.py` — Tests that exercise the metadata module through the shim path; useful for understanding the public API contract
- [repo] `pyproject.toml` — Check whether both `reasons` and `reasons_lib` are declared as packages, and whether there are any deprecation warnings or `pip`-level aliases configured

## Beliefs

- `reasons-lib-metadata-is-a-reason-metadata` — `reasons_lib/metadata.py` re-exports every public name from `reasons.metadata` and defines nothing of its own; it is a strict alias, not a superset.
- `star-import-prevents-name-filtering` — Because the shim uses `import *`, any name added to `reasons.metadata` automatically appears in `reasons_lib.metadata` without touching the shim file; conversely, no name can be hidden or renamed here.
- `nine-consumers-still-use-shim` — As of this code state, 9 files (7 in `reasons/`, 2 in `tests/`) still import via `reasons_lib.metadata`, meaning the deprecation is not yet complete and removing this file would break those importers.
- `shim-has-no-runtime-logic` — The module body contains no function calls, conditionals, or side effects beyond the star import; it cannot raise domain-specific errors, only standard `ImportError`/`AttributeError` from name resolution.
