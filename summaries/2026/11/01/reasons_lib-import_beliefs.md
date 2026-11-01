---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/import_beliefs.py

# `reasons_lib/import_beliefs.py`

## Purpose

This file is a **one-line backwards-compatibility shim**. It exists solely so that any code still using the old import path `reasons_lib.import_beliefs` continues to work after the project's canonical package was renamed (or restructured) to `reasons`. It owns no logic, no state, and no behavior of its own — it is a pure re-export bridge.

If you were onboarding and found `import reasons_lib.import_beliefs` in an old ticket, a downstream consumer, or a test written before the rename, this file is why it hasn't broken.

## Key Components

There is exactly one executable statement:

```python
from reasons.import_beliefs import *  # noqa: F401,F403
```

- **`import *`** pulls every public name (no leading underscore, and everything listed in `__all__` if defined) from the real module into the `reasons_lib.import_beliefs` namespace.
- **`# noqa: F401,F403`** silences two linter warnings:
  - `F401` – "imported but unused" (the star import is intentionally "unused" in this file).
  - `F403` – "star import makes it impossible to tell what names are exported."
- The **docstring** is the only documentation and explicitly tells the reader: *"use `from reasons.import_beliefs` instead."*

There are no classes, no functions, no constants, and no `__all__` override. Whatever `reasons.import_beliefs` exports, this module exports.

## Patterns

- **Package-rename shim / namespace bridge.** The entire `reasons_lib/` directory mirrors the `reasons/` directory file-for-file. Every file in `reasons_lib/` follows this same two-line pattern. This is the standard "keep the old import path alive during a rename" idiom (analogous to what CPython did when moving modules between top-level packages, or what Django did when renaming internal apps).
- **Star-import re-export.** Rather than listing each symbol explicitly (`from reasons.import_beliefs import import_beliefs, deduplicate, ...`), the shim uses `*`. This means if a new public function is added to `reasons.import_beliefs`, the shim picks it up automatically with zero maintenance. The trade-off is reduced discoverability, which the `noqa` comments acknowledge.
- **Zero-abstraction.** No wrapper functions, no deprecation warnings, no logging. The shim is intentionally invisible — it should behave *exactly* as if the caller imported from `reasons` directly.

## Dependencies

| Direction | Modules |
|-----------|---------|
| **This file imports** | `reasons.import_beliefs` (the real implementation) |
| **Imported by** (within this repo) | `reasons/api.py`, `reasons/import_agent.py`, `reasons/pg.py`, and several tests: `test_import_beliefs.py`, `test_metadata.py`, `test_metadata_extended.py`, `test_nogood_id.py`, `test_roundtrip.py`, `test_source_type.py` |

The fact that modules *inside* `reasons/` still import through `reasons_lib.import_beliefs` (e.g., `reasons/api.py`) is a small code smell — it means the internal rename is not yet fully complete. Those internal imports should eventually be updated to `reasons.import_beliefs` directly, after which the `reasons_lib/` package can be deleted.

## Flow

There is no runtime flow beyond Python's import machinery:

1. Something executes `import reasons_lib.import_beliefs` (or `from reasons_lib.import_beliefs import X`).
2. Python resolves `reasons_lib` → finds `reasons_lib/__init__.py`, then `reasons_lib/import_beliefs.py`.
3. That module executes its single `from reasons.import_beliefs import *` statement.
4. Python resolves `reasons.import_beliefs`, executes the *real* module (which defines the actual functions, classes, constants), and copies every public name into the `reasons_lib.import_beliefs` namespace dict.
5. The caller's `X` now resolves to the same object that `reasons.import_beliefs.X` would.

No data transformation, no side effects, no I/O.

## Invariants

- **Transparency:** `reasons_lib.import_beliefs.S` is the *same object* as `reasons.import_beliefs.S` (not a copy, not a wrapper). `is` identity is preserved.
- **Completeness:** Every public symbol in `reasons.import_beliefs` is reachable via `reasons_lib.import_beliefs`. Adding a new public function to the real module automatically makes it available through the shim.
- **No private leakage:** Symbols prefixed with `_` are *not* re-exported (standard `import *` semantics), assuming `reasons.import_beliefs` doesn't define `__all__` to override this.

## Error Handling

There is **none** in this file. All error behavior is inherited from `reasons.import_beliefs`:

- If `reasons.import_beliefs` itself fails to import (e.g., a missing dependency like `pg` or `storage`), the `ImportError` / `ModuleNotFoundError` propagates to the caller of `reasons_lib.import_beliefs` unchanged.
- Any runtime errors raised by the functions defined in `reasons.import_beliefs` (e.g., validation failures during belief import) propagate normally through the caller's call stack.
- There are no `try/except` blocks, no fallbacks, and no swallowed exceptions.

---

## Topics to Explore

- [file] `reasons/import_beliefs.py` — The actual implementation: understand what beliefs are, how they're structured, and what the import pipeline does (deduplication, validation, network insertion).
- [file] `reasons_lib/__init__.py` — Confirm whether the package-level init does anything beyond being a namespace, and whether it also re-exports or just lets submodules resolve naturally.
- [general] `reasons-lib-to-reasons-rename` — The broader migration: how many `reasons_lib` imports remain in the codebase, whether a deprecation timeline exists, and what the `CHANGELOG.md` says about the rename.
- [file] `tests/test_import_beliefs.py` — The test suite exercises the import path; check whether it imports from `reasons` or `reasons_lib`, and what edge cases (dedup, contradictions, metadata) are covered.
- [file] `reasons/api.py` — This module imports `reasons_lib.import_beliefs` internally, making it a key consumer to update when the shim is eventually removed.

## Beliefs

- `reasons-lib-import-beliefs-is-a-reasons-import-beliefs` — `reasons_lib.import_beliefs` is a pure star-import re-export of `reasons.import_beliefs`; no function, class, or constant is defined locally, and `is`-identity holds for every shared symbol.
- `reasons-lib-shim-requires-reasons-module` — The shim has a hard import-time dependency: if `reasons.import_beliefs` raises (missing dependency, syntax error), `import reasons_lib.import_beliefs` fails with the same exception; there is no fallback or graceful degradation.
- `reasons-internal-modules-still-import-reasons-lib` — At least three modules inside the `reasons/` package (`api.py`, `import_agent.py`, `pg.py`) still route through `reasons_lib.import_beliefs` rather than importing `reasons.import_beliefs` directly, indicating the internal rename is incomplete.
- `reasons-lib-star-import-produces-auto-propagation` — Because the shim uses `import *` rather than an explicit symbol list, any new public name added to `reasons.import_beliefs` becomes immediately available through `reasons_lib.import_beliefs` without touching the shim file.
