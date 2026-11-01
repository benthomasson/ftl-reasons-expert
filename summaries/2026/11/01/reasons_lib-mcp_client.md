---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons_lib/mcp_client.py

# `reasons_lib/mcp_client.py`

## Purpose

This file is a **backwards-compatibility shim** — a one-line re-export that preserves the old import path (`reasons_lib.mcp_client`) after the package was renamed or restructured to `reasons`. Its sole responsibility is to let existing code that does `from reasons_lib.mcp_client import X` continue to work without modification. The docstring explicitly tells developers the canonical location is now `reasons.mcp_client`.

This is part of a broader migration: the entire `reasons_lib/` directory mirrors `reasons/` file-for-file (every module present in `reasons/` has a twin in `reasons_lib/`), indicating a bulk package rename from `reasons_lib` → `reasons`.

## Key Components

There are no classes, functions, or constants defined here. The file contains exactly one operative statement:

```python
from reasons.mcp_client import *  # noqa: F401,F403
```

- **`import *`** — pulls every public name (i.e., names not starting with `_`) from `reasons.mcp_client` into this module's namespace. Whatever `reasons.mcp_client` exports (classes like an MCP client wrapper, factory functions, type aliases, constants), they become available as `reasons_lib.mcp_client.<name>`.
- **`# noqa: F401,F403`** — suppresses two `flake8`/`ruff` lint rules:
  - `F401` ("imported but unused") — the star import is intentional, not a mistake.
  - `F403` ("star imports make it unknown which names are defined") — the project accepts the ambiguity in this shim because it's a compatibility layer, not a source of truth.

## Patterns

- **Re-export / alias module**: A well-known Python idiom for maintaining import-path stability across a package rename. The shim module owns no logic; it delegates entirely to the canonical module.
- **`reasons_lib/` as a compatibility layer**: The parallel directory structure (`reasons/` vs. `reasons_lib/`) is a transitional pattern. Consumers can migrate at their own pace. Eventually `reasons_lib/` will be deleted.
- **Star import over explicit re-export**: The project chose `import *` rather than listing each name (`from reasons.mcp_client import McpClient, create_client, ...`). This means the shim **automatically tracks** additions to `reasons.mcp_client` without needing a manual update. The trade-off is that `from reasons_lib.mcp_client import *` is fragile for type checkers and IDE autocompletion.

## Dependencies

| Direction | Module | Relationship |
|-----------|--------|--------------|
| **Imports** | `reasons.mcp_client` | Re-exports all its public names. This is the sole hard dependency. |
| **Imported by** | `reasons/cli.py` | Still uses the `reasons_lib.mcp_client` path (or a `reasons_lib`-prefixed path), meaning the CLI has not yet been migrated to the new package name. |

> Note: `reasons/cli.py` importing from `reasons_lib` is a bit unusual — it suggests the CLI module itself may be in the process of migration, or that `reasons_lib` is treated as the "stable" import surface for internal tooling while `reasons` is the "new" surface for external consumers.

## Flow

Execution is trivial:

1. Python's import machinery resolves `reasons_lib.mcp_client`.
2. The module body executes the single star import.
3. Python imports `reasons.mcp_client` (executing its full module body — classes, functions, etc.).
4. All public names from `reasons.mcp_client` are copied into `reasons_lib.mcp_client`'s `__dict__`.
5. Any subsequent `from reasons_lib.mcp_client import SomeName` simply looks up `SomeName` in this namespace.

No runtime logic, no side effects beyond the initial import of the target module.

## Invariants

- **Namespace parity**: Every public name in `reasons.mcp_client` is guaranteed to be accessible via `reasons_lib.mcp_client`. If someone adds a new public symbol to `reasons.mcp_client`, it immediately becomes available through the shim with zero changes to this file.
- **No new public names**: This module should never define its own classes, functions, or constants. Adding logic here would break the "thin alias" contract and create two sources of truth.
- **Deletion safety**: Once all consumers (including `reasons/cli.py`) have migrated to `reasons.mcp_client`, the entire `reasons_lib/` directory can be removed. No data or state is owned by this shim.

## Error Handling

Error handling is entirely **delegated to the import system**:

- If `reasons.mcp_client` doesn't exist (e.g., the canonical module was deleted), Python raises `ModuleNotFoundError` at import time. There is no try/except, no fallback.
- If `reasons.mcp_client` itself has an import error (missing dependency, syntax error), that exception propagates up through this module's import.
- There is **no** custom error catching, logging, or graceful degradation. The shim is either fully functional or the import fails hard.

This is appropriate for a shim — you want failures to be loud and immediate so developers notice broken import paths during migration rather than hitting subtle runtime `AttributeError`s later.

## Topics to Explore

- [file] `reasons/mcp_client.py` — The canonical implementation this shim delegates to; understanding its public API surface tells you exactly what `reasons_lib.mcp_client` re-exports
- [file] `reasons/cli.py` — The sole known consumer of this shim; reading its imports reveals whether it uses the old or new path and what MCP client functionality the CLI relies on
- [general] `reasons-lib-migration` — The broader package-rename from `reasons_lib` to `reasons`; check `CHANGELOG.md` and `pyproject.toml` for the migration timeline and any deprecation schedule
- [file] `reasons_lib/__init__.py` — Whether the `reasons_lib` package `__init__` adds any additional re-exports or version checks that affect how submodules are resolved
- [file] `tests/test_mcp_server.py` — The MCP server test suite; understanding the server side helps contextualize what the client wrapper (and thus this shim) exposes

## Beliefs

- `reasons-lib-mcp-client-is-a-shim-for-reasons-mcp-client` — `reasons_lib/mcp_client.py` defines zero logic of its own; its entire contract is to re-export the public namespace of `reasons.mcp_client`.
- `star-import-tracks-canonical-additions` — Because the shim uses `import *`, any new public symbol added to `reasons.mcp_client` is automatically available through `reasons_lib.mcp_client` without modifying the shim.
- `reasons-cli-py-still-consumes-reasons-lib-path` — `reasons/cli.py` is the only known importer of this shim, indicating the CLI has not yet been fully migrated to the `reasons.*` import namespace.
- `reasons-lib-directory-is-transitional` — The `reasons_lib/` package is a deletion candidate; it owns no unique code and exists solely to preserve old import paths during the `reasons_lib` → `reasons` rename.
- `no-error-swallowing-in-shim` — The shim contains no try/except blocks; any import failure in `reasons.mcp_client` propagates as an unhandled exception, ensuring broken paths fail loudly at import time.
