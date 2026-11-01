---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/mcp_server.py

# `reasons_lib/mcp_server.py`

## Purpose

This file is a **backwards-compatibility shim** — a one-line re-export that exists solely so that any code still importing from the old `reasons_lib` namespace continues to work. The actual MCP server implementation lives in `reasons/mcp_server.py`. The docstring makes the intent explicit: *"use `from reasons.mcp_server` instead."*

This is part of a broader structural migration visible in the repo: every module in `reasons/` has a corresponding file in `reasons_lib/`, and the `reasons_lib` versions are almost certainly all shims like this one. The project was originally packaged under the `reasons_lib` name and has since been refactored to the shorter `reasons` package name.

## Key Components

There are no classes, functions, or constants defined here. The entire file is:

```python
"""Backwards-compatibility shim — use ``from reasons.mcp_server`` instead."""
from reasons.mcp_server import *  # noqa: F401,F403
```

- **`import *`** — re-exports every public name (anything not prefixed with `_`) from `reasons.mcp_server` into the `reasons_lib.mcp_server` namespace. The set of available names is entirely determined by the source module's `__all__` (if defined) or its module-level public attributes.
- **`# noqa: F401,F403`** — suppresses `flake8`/`ruff` warnings:
  - **F401**: "imported but unused" (expected, since the star import is the point)
  - **F403**: "star import makes names unavailable for static lookup" (accepted trade-off for a shim)

## Patterns

- **Delegation / Facade**: This is a classic facade pattern where a thin layer forwards all calls to the real implementation. The caller gets the same API surface as `reasons.mcp_server` but through the legacy `reasons_lib` path.
- **Star-import re-export**: The idiomatic Python pattern for "I don't want to enumerate every symbol; just mirror the namespace." This is fragile in the sense that if `reasons.mcp_server` renames or removes a public symbol, the shim silently loses it — but for a transitional shim that's acceptable.
- **Shim directory convention**: The entire `reasons_lib/` package follows this pattern. It's a parallel namespace that mirrors `reasons/` one-to-one. This is a common migration strategy: ship the new package, keep the old import path working for one or two release cycles, then delete the shims.

## Dependencies

| Direction | Module | Notes |
|---|---|---|
| **Imports (this →)** | `reasons.mcp_server` | The single, real implementation. All names are pulled from here. |
| **Imported by (→ this)** | `tests/test_mcp_server.py` | The test suite still uses the `reasons_lib` path, which is why the shim must remain. |

No other production code imports this file directly (the rest of the repo imports from `reasons.*`).

## Flow

Execution is trivial and linear:

1. Python's import machinery loads `reasons_lib.mcp_server`.
2. It executes `from reasons.mcp_server import *`, which triggers loading of `reasons/mcp_server.py` (if not already in `sys.modules`).
3. All public names from `reasons.mcp_server` are copied into the `reasons_lib.mcp_server` module's `__dict__`.
4. Any subsequent `from reasons_lib.mcp_server import X` resolves `X` against that `__dict__` and returns the same object that `reasons.mcp_server.X` would return.

There is no runtime logic, no state, no side effects beyond the import itself.

## Invariants

- **Name parity**: Every public name accessible via `reasons_lib.mcp_server` must also be a public name in `reasons.mcp_server`. The shim enforces this implicitly by virtue of the star import — if a name is missing from the source, it simply isn't available in the shim.
- **No independent logic**: This file must never define its own functions, classes, or mutable state. Doing so would break the "thin facade" contract and create a second source of truth.
- **One-way dependency**: `reasons_lib` depends on `reasons`, never the reverse. The shim must not import from `reasons_lib` back.

## Error Handling

There is no error-handling code in this file. Errors propagate naturally:

- **Import-time**: If `reasons.mcp_server` fails to import (e.g., missing dependency, syntax error), the `from ... import *` line raises `ImportError` or `ModuleNotFoundError` at module-load time. The caller of `reasons_lib.mcp_server` sees the same exception.
- **Runtime**: Once imported, any function called through the shim is the *same object* as in `reasons.mcp_server`, so all exception types, semantics, and stack traces are identical. The shim adds no try/except, no logging, no wrapping.

The only "error handling" is the `noqa` comments, which suppress static-analysis false positives rather than handling runtime errors.

---

## Topics to Explore

- [file] `reasons/mcp_server.py` — The actual MCP server implementation; all real logic (server setup, tool registration, request handling) lives here. This is where the file you're looking at delegates everything.
- [file] `reasons_lib/__init__.py` — Check whether the entire `reasons_lib` package is a shim directory or if some modules contain real code. This determines whether `reasons_lib` can be deleted in one pass.
- [file] `tests/test_mcp_server.py` — The sole consumer of this shim. Understanding what it imports and asserts will tell you whether you can rewrite it to use `reasons.mcp_server` and then delete this file.
- [general] `reasons-namespace-migration` — The repo has a near-1:1 file mapping between `reasons/` and `reasons_lib/`. Understanding the full scope (which shims exist, which are still needed, which are dead) is key to planning the cleanup.
- [file] `pyproject.toml` — Check the `packages` / `find` configuration to confirm whether both `reasons` and `reasons_lib` are declared as installable packages, or if only one is. This affects whether downstream consumers can still resolve the `reasons_lib` path.

---

## Beliefs

- `reasons-lib-mcp-server-is-a-thin-facade` — `reasons_lib/mcp_server.py` defines zero logic; every public name it exposes is the identical object exported by `reasons/mcp_server.py`.
- `reasons-lib-mcp-server-requires-reasons-mcp-server` — This file cannot be imported successfully unless `reasons.mcp_server` is importable and resolvable; it has no fallback or conditional import path.
- `tests-test-mcp-server-imports-reasons-lib` — `tests/test_mcp_server.py` is the only known consumer that imports via the `reasons_lib` path, making it the gating dependency for deleting this shim.
- `reasons-lib-replaces-reasons` — The `reasons_lib` package is a legacy namespace that predates `reasons`; the project's canonical import path is now `reasons.*`, and `reasons_lib` exists purely for backwards compatibility.
- `star-import-noqa-f403-accepted` — The `# noqa: F403` on the star import is an intentional, project-level acceptance of the trade-off (static-analysis opacity) in exchange for zero-maintenance re-exports; this is a convention repeated across all `reasons_lib` shims.
