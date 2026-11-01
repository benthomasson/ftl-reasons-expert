---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/report_belief.py

# `reasons_lib/report_belief.py`

## Purpose

This file is a **backwards-compatibility shim**. It exists solely so that any code (external libraries, downstream projects, or internal modules) that still imports from the legacy `reasons_lib` package namespace can continue to resolve the `report_belief` module without modification. The canonical implementation lives in `reasons/report_belief.py`; this file is a zero-logic re-export.

You can think of it as a "tombstone" file during a package rename: the project was restructured from `reasons_lib` → `reasons`, and these thin wrappers prevent a breaking change for consumers who haven't updated their `import` statements yet.

## Key Components

There are **no** classes, functions, constants, or logic in this file. The entire module body is:

```python
from reasons.report_belief import *  # noqa: F401,F403
```

- The `*` star-import pulls every public name (no leading underscore) defined in `reasons.report_belief` into this module's namespace.
- `# noqa: F401` suppresses the "imported but unused" linter warning (the import *is* the point).
- `# noqa: F403` suppresses the "wildcard import" style warning.
- Because it's a star-import, any name added to `reasons.report_belief` automatically becomes available here — no list to maintain.

## Patterns

- **Adapter / Facade pattern (degenerate form):** The module acts as a transparent pass-through. A caller doing `from reasons_lib.report_belief import report_belief` gets the exact same object as `from reasons.report_belief import report_belief`.
- **Star-import re-export:** A common Python idiom for compatibility shims. It avoids an explicit `__all__` list and keeps the shim in sync with the source module automatically.
- **Dual-package layout:** The repo maintains two top-level packages (`reasons/` and `reasons_lib/`) with near-identical file trees. Every file in `reasons_lib/` is presumably a shim like this one, pointing back to the `reasons/` equivalent. This lets the project deprecate the old name gracefully.

## Dependencies

| Direction | Module | Relationship |
|-----------|--------|--------------|
| **This file imports** | `reasons.report_belief` | Star-import; source of truth for all public symbols |
| **`reasons/api.py` imports this** | — | Likely does `from reasons_lib.report_belief import …` for a backward-compat code path, or tests that both namespaces resolve identically |
| **`tests/test_report_belief.py` imports this** | — | Verifies the shim works (i.e., that `reasons_lib.report_belief.X` is the same object as `reasons.report_belief.X`) |

No third-party dependencies. No I/O. No side effects beyond module-level name binding.

## Flow

Execution is trivial:

1. Python's import machinery loads `reasons_lib/__init__.py`, then `reasons_lib/report_belief.py`.
2. It encounters the single `from reasons.report_belief import *` statement.
3. Python imports `reasons.report_belief` (the real implementation), then copies every public name into `reasons_lib.report_belief.__dict__`.
4. Any subsequent `from reasons_lib.report_belief import <name>` resolves to the same function/class object that `reasons.report_belief` exposes.

There is no branching, no state mutation, no lazy loading. It is a pure alias.

## Invariants

- **Identity guarantee:** For every public name `N` in `reasons.report_belief`, `reasons_lib.report_belief.N is reasons.report_belief.N` must be `True`. The star-import of function/class objects (not copies) enforces this.
- **No new public names:** This module must not define its own top-level names. Any new symbol added here would diverge the two namespaces and defeat the purpose of the shim.
- **No circular import risk:** `reasons.report_belief` must not (transitively) import from `reasons_lib`. If it did, you'd get a circular-import error at module load time.

## Error Handling

There is none here. Errors are entirely those of `reasons.report_belief`:

- If `reasons.report_belief` fails to import (missing dependency, syntax error), the failure surfaces at the `from … import *` line in this file. The traceback will show both frames.
- If a caller requests a name that doesn't exist in `reasons.report_belief`, they get a standard `ImportError` — identical to importing from `reasons.report_belief` directly.
- No exceptions are caught, suppressed, or re-wrapped.

## Topics to Explore

- [file] `reasons/report_belief.py` — The actual implementation this shim re-exports; understand the `report_belief` function's contract, parameters, and return type.
- [file] `reasons_lib/__init__.py` — Check whether the package init does anything (version pinning, sub-package registration) or is itself a shim.
- [file] `tests/test_report_belief.py` — See how the test suite validates the shim, particularly whether it asserts identity (`is`) between the two namespaces.
- [general] `package-rename-strategy` — How the project handles the `reasons_lib` → `reasons` transition: is there a deprecation timeline, a `__getattr__`-based lazy shim elsewhere, or is this static star-import the only mechanism?
- [file] `pyproject.toml` — Confirm which top-level package name is the "installed" one (e.g., `[project] name = "ftl-reasons"`) and whether both `reasons` and `reasons_lib` are listed as packages for distribution.

## Beliefs

- `reasons-lib-report-belief-is-a-re-export-of-reasons-report-belief` — Every public name accessible via `reasons_lib.report_belief` is the identical Python object exported by `reasons.report_belief` (identity, not equality).
- `shim-star-import-prevents-drift` — Because the shim uses `import *` rather than an explicit name list, adding a new public function to `reasons.report_belief` automatically makes it available in `reasons_lib.report_belief` without touching the shim file.
- `reasons-lib-namespace-must-not-introduce-symbols` — The shim defines no local names of its own; any new top-level symbol in `reasons_lib/report_belief.py` would violate the "transparent alias" contract and could mask a rename in the canonical module.
- `import-of-reasons-lib-report-belief-triggers-import-of-reasons-report-belief` — Loading the shim is sufficient to fully execute `reasons.report_belief`'s module-level code (side effects, global state initialization), so the shim cannot be used to *avoid* loading the implementation.
