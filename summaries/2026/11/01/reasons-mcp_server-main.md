---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# Function: main in reasons/mcp_server.py

# `main()` in `reasons/mcp_server.py`

## Purpose

`main()` is the **entry point** for the MCP (Model Context Protocol) server process. It does two things before handing control to the framework:

1. **Resolves the database path eagerly** by calling `_find_db()` and storing it in the module-level `_db` global.
2. **Starts the FastMCP server loop** via `mcp.run()`, which blocks and handles JSON-RPC messages over stdio until the client disconnects.

It exists so that the `reasons-mcp` console script (defined in `pyproject.toml`) and the `if __name__ == "__main__"` guard both funnel through a single, testable function.

## Contract

| Aspect | Detail |
|---|---|
| **Preconditions** | The `mcp` package must be importable (checked at module load time — if it's missing, an `ImportError` is raised before `main()` is ever reachable). The `reasons.api` module must be importable. The process must have a working stdio channel (the MCP transport). |
| **Postconditions** | `_db` is a non-`None` `str` pointing to a valid `reasons.db` path (or the `api.DEFAULT_DB` fallback). The MCP server is running and servicing requests until the stdio stream closes. |
| **Invariants** | Every `@mcp.tool()` handler in this module calls `_get_db()`, which checks `_db is None` before falling back to `_find_db()`. Because `main()` pre-assigns `_db`, the lazy path in `_get_db()` is effectively dead code in the normal server flow — but it's there as a safety net if someone imports the module and calls a tool handler in a test without going through `main()`. |

## Parameters

**None.** The function takes no arguments. All configuration is resolved from:

- The `REASONS_DB` environment variable (checked first inside `_find_db()`).
- The current working directory and its ancestors (filesystem walk in `_find_db()`).
- The hardcoded `api.DEFAULT_DB` constant (last resort).

This means the caller's responsibility is entirely about **how the process is launched** (env var, cwd), not about passing arguments to `main()` itself.

## Return Value

`main()` returns whatever `mcp.run()` returns. In practice, `mcp.run()` is a **blocking call** that runs the stdio event loop and only returns (typically `None`) when the client closes the pipe or sends a shutdown signal. There is no meaningful return value to consume — the function is designed to be called at process termination.

## Algorithm

Step-by-step:

1. **`global _db`** — declares that assignments to `_db` inside this function target the module-level variable (not a local). Without this line, `_db = _find_db()` would create a local variable and the module-level `_db` would remain `None`, breaking every tool handler.

2. **`_db = _find_db()`** — resolves the database path:
   - If `REASONS_DB` is set in the environment, return that path immediately.
   - Otherwise, walk up from `os.getcwd()` checking for `reasons.db` at each directory level, until the filesystem root is reached.
   - If no `reasons.db` is found anywhere, fall back to `api.DEFAULT_DB`.
   
   The result is stored in the module-level `_db`.

3. **`mcp.run()`** — starts the FastMCP server. This internally:
   - Registers all functions decorated with `@mcp.tool()` as callable tools.
   - Opens a stdio-based JSON-RPC loop (read line → parse request → dispatch to tool → write response).
   - Blocks indefinitely until the stdio stream is closed or a signal is received.

## Side Effects

- **Module state mutation**: `_db` (a module-level global) transitions from `None` to a concrete path string. This is visible to every other function in the module.
- **Filesystem I/O**: `_find_db()` calls `os.getcwd()`, `os.path.join()`, `os.path.exists()`, and `os.path.dirname()` in a loop. This is read-only but touches the filesystem.
- **Network / IPC**: `mcp.run()` opens a stdio reader/writer and begins a long-running event loop. This is the dominant side effect — the process is now a server.
- **No database writes**: At this point `main()` does not open a connection to the SQLite/Postgres database. Actual DB access is deferred to individual tool handler calls via `api.*` functions.

## Error Handling

`main()` itself has **no try/except**. Errors surface as follows:

| Failure point | Behavior |
|---|---|
| `mcp` package missing | `ImportError` is raised at **import time** (top of file), before `main()` is ever called. The message directs the user to `pip install 'ftl-reasons[mcp]'`. |
| `_find_db()` | Never raises. Worst case it returns `api.DEFAULT_DB`. However, if the path it returns doesn't actually point to a valid database, the failure will surface later when the first tool handler calls `api.search(...)`, `api.show_node(...)`, etc. — those will raise `sqlite3.OperationalError` or `KeyError`, which are caught by the individual `@mcp.tool()` wrappers and returned as JSON error objects. |
| `mcp.run()` | If the stdio transport can't be established (e.g., no stdin/stdout), this will raise. In practice, this is a process-level failure that crashes the server. |

**Key assumption not enforced by the type system**: `_find_db()` returns a `str`, but there is no guarantee the file at that path is a valid, openable database. The `str | None` type on `_db` documents that it *can* be `None`, but after `main()` runs it is always a `str` — the type annotation is broader than the runtime guarantee.

## Usage Patterns

```bash
# Typical invocation via console script (defined in pyproject.toml):
reasons-mcp

# With explicit DB path:
REASONS_DB=/path/to/reasons.db reasons-mcp

# Direct Python invocation (less common):
python -m reasons.mcp_server
```

The `if __name__ == "__main__": main()` guard at the bottom of the file supports the direct invocation. In the MCP ecosystem, the server is spawned as a **child process** by the client (e.g., Claude Code, VS Code). The client pipes JSON-RPC messages to stdin and reads responses from stdout. The process lifecycle is:

```
client spawns `reasons-mcp`
  → main() runs
    → _db resolved
    → mcp.run() blocks, servicing requests
  → client closes stdin
    → mcp.run() returns
  → process exits
```

The caller (the MCP client) is responsible for:
- Setting `REASONS_DB` or ensuring cwd is in the right directory tree.
- Closing the stdio pipe to shut down the server gracefully.
- Not expecting any output on stdout other than JSON-RPC frames (the server must not `print()` anything else).

## Dependencies

| Dependency | Role |
|---|---|
| `mcp.server.fastmcp.FastMCP` | External. Provides the `mcp` server instance and the `@mcp.tool()` decorator. `mcp.run()` is the blocking server loop. |
| `reasons.api` | Internal. All tool handlers delegate to `api.*` functions. `main()` itself only references `api.DEFAULT_DB` (via `_find_db()`'s fallback). |
| `os` | Standard library. Used by `_find_db()` for path resolution and environment variable lookup. |
| `_find_db()` (same module) | Resolves the DB path. Called once by `main()`, and lazily by `_get_db()` if `_db` is still `None`. |
| `_get_db()` (same module) | Not called by `main()` directly, but is the access pattern all tool handlers use to get the DB path. `main()`'s job is to make `_db` non-`None` so that `_get_db()` short-circuits. |

## Topics to Explore

- [function] `reasons/mcp_server.py:_find_db` — The directory-walking DB discovery logic that `main()` delegates to; understanding its fallback chain and the `REASONS_DB` env var contract is essential to debugging "wrong database" issues.
- [function] `reasons/mcp_server.py:_get_db` — The lazy initialization guard that `main()` effectively bypasses; important to understand why both eager and lazy paths exist (test vs. production).
- [file] `tests/test_mcp_server.py` — Shows how tool handlers are invoked in tests (likely without calling `main()`), which reveals the design intent behind the `_get_db()` lazy fallback.
- [general] `mcp-stdio-transport` — How the Model Context Protocol uses stdin/stdout as the wire format; understanding the message framing and lifecycle (spawn → serve → close) explains why `mcp.run()` is a blocking call and why the process must stay alive.
- [file] `reasons/api.py` — The core API layer that every tool handler calls; understanding its function signatures, error semantics, and `DEFAULT_DB` constant gives context for what happens after `main()` hands control to the tool dispatch loop.

## Beliefs

- `main-pre-resolves-db-path` — `main()` calls `_find_db()` eagerly and assigns the result to the module-level `_db` global, making the lazy `if _db is None` branch inside `_get_db()` unreachable in the normal server lifecycle (it only matters when tool handlers are called directly in tests).
- `mcp-server-uses-stdio-not-tcp` — The MCP server in this module communicates exclusively over stdio (spawned as a child process by the client), not over a network socket; `mcp.run()` blocks on the stdin stream and there is no port or host configuration.
- `main-delegates-db-discovery-to-environment` — The database path is determined entirely by the `REASONS_DB` environment variable or a filesystem walk from cwd; `main()` itself accepts no parameters, so all configuration is ambient to the process environment.
- `tool-handlers-catch-exceptions-into-json` — Every `@mcp.tool()` function wraps its `api.*` call in a try/except and returns a JSON-encoded `{"error": "..."}` dict rather than raising, meaning the MCP client always receives a well-formed JSON response even on failure — a contract that `main()` must preserve by not interfering with the stdio stream.
- `import-error-guards-mcp-optional` — The `mcp` package is an optional dependency; the module raises a descriptive `ImportError` at import time if it's missing, which means `main()` is unreachable in that case and the error is never a runtime failure inside `main()` itself.
