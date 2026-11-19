---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: main in reasons/mcp_server.py

# Explanation: `main` in `reasons/mcp_server.py`

## Purpose

`main` is the process entry point for the `ftl-reasons` MCP (Model Context Protocol) server. It exists to:

1. Resolve the SQLite database path **eagerly** at startup (so failures/misconfiguration surface before the server loop begins, and so every tool call sees the same resolved path).
2. Start the `FastMCP` server, which communicates over stdio and exposes the many `@mcp.tool()`-decorated functions (`search`, `show`, `add`, `retract`, etc.) to an MCP client such as Claude Code.

It is a tiny function — ~3 lines — but it wires together the module-level `mcp` instance with the resolved database location.

## Contract

- **Preconditions**
  - The `mcp` package must be importable; otherwise the module-level import block raises `ImportError` *before* `main` is even defined (`raise ImportError("The 'mcp' package is required...")`).
  - The module-level `mcp = FastMCP("reasons")` must have succeeded.
- **Postconditions**
  - The module global `_db` holds a non-`None` string path (as soon as `_find_db()` returns).
  - `mcp.run()` has been invoked and, on normal return, the server has stopped listening.
- **Invariants**
  - After `main` runs, `_get_db()` will return the eagerly-resolved value without re-running discovery (because `_get_db` short-circuits on `_db is not None`).
  - The resolved path is used consistently by every tool for the lifetime of the server, since every tool calls `_get_db()`.

## Parameters

None. `main` takes zero arguments. It reads its configuration entirely from:

- `os.environ["REASONS_DB"]` (optional override, handled inside `_find_db`).
- The current working directory (`os.getcwd()`), also handled inside `_find_db`.

## Return Value

Returns `None` (implicitly). In practice, `mcp.run()` is a **blocking** call — it runs the stdio transport loop until the client disconnects or the process is terminated by signal. So `main` typically never returns during normal operation.

## Algorithm

Step-by-step:

1. **`global _db`** — declare that the assignment in this function targets the module-level `_db`, not a local.
2. **`_db = _find_db()`** — resolve the DB path:
   - If `REASONS_DB` env var is set → return it verbatim.
   - Else walk from `os.getcwd()` upward, looking for a sibling `reasons.db`; return the first hit.
   - Else if the filesystem root is reached → return `api.DEFAULT_DB`.
3. **`mcp.run()`** — hand control to `FastMCP.run()`, which starts the JSON-RPC server on stdio. Every decorated `@mcp.tool()` function becomes callable by the MCP client. Because `_db` is already set, tools do not re-trigger discovery via `_get_db`.

## Side Effects

- **Global mutation**: assigns to module-level `_db`.
- **Environment access**: `_find_db` reads `REASONS_DB`.
- **Filesystem access**: `_find_db` calls `os.getcwd()` and `os.path.exists()` for each ancestor directory.
- **Process I/O**: `mcp.run()` binds to stdin/stdout and speaks the MCP protocol. It blocks the calling thread.
- **No database connection is opened by `main` itself** — the DB is only opened lazily when a tool actually invokes an `api.*` call.

## Error Handling

`main` has **no try/except block**. Failures propagate:

- **`ImportError`** — raised at module import time (not in `main`) if `mcp` is missing. This is why the message tells users to `pip install 'ftl-reasons[mcp]'`.
- **`_find_db` does not raise** on a missing DB; it falls back to `api.DEFAULT_DB`. So "database not found" is not an error here — the downstream `api` layer decides whether that's fatal.
- **`mcp.run()` exceptions** (transport errors, broken pipes on stdin, etc.) are not swallowed. They propagate to the caller of `main` (usually the interpreter exit path).
- **No validation** that the returned `_db` path actually points to an existing, readable file.

## Usage Patterns

Two entry points:

- **Module execution**: `python -m reasons.mcp_server` hits `if __name__ == "__main__": main()`.
- **Console script**: The package presumably exposes `reasons-mcp` as an entry point that resolves to `reasons.mcp_server:main`.

Typical wiring in an MCP client config:

```json
{
  "mcpServers": {
    "reasons": { "command": "reasons-mcp", "env": { "REASONS_DB": "/path/reasons.db" } }
  }
}
```

**Caller obligations**: call it once, in a process whose stdin/stdout are dedicated to the MCP transport (do not share stdout with other logging — anything printed to stdout will corrupt the protocol stream).

## Dependencies

- **Internal**: `_find_db` (the discovery helper), `_get_db` (indirectly, since it shares `_db`), and `api.DEFAULT_DB` (fallback via `_find_db`).
- **External**: `mcp.server.fastmcp.FastMCP` (the `mcp` package).
- **Stdlib**: (indirectly through `_find_db`) `os`.

## Assumptions Not Enforced by the Type System

- `_find_db()` returns `str`, but the file it names may not exist — nothing in `main` validates this. Any "cannot open database" error will surface later inside an `api.*` call from a tool handler.
- `mcp.run()` is assumed to be blocking and non-reentrant. Calling `main()` twice in the same process would re-assign `_db` and then re-enter `mcp.run()`, which is unlikely to be supported.
- stdout/stderr are assumed to belong to the MCP transport. `main` does not guard against print statements or logging to stdout elsewhere in the process.
- The environment variable convention `REASONS_DB` is assumed consistent with the CLI entry point (`reasons`), but `main` itself has no way to verify the CLI and MCP paths resolve the same DB.

## Topics to Explore

- [function] `reasons/mcp_server.py:_find_db` — The discovery algorithm `main` relies on: env var → walk up → `api.DEFAULT_DB` fallback
- [function] `reasons/mcp_server.py:_get_db` — Lazy counterpart that `main`'s eager assignment short-circuits; explains the shared-state contract
- [file] `tests/test_mcp_server.py` — Tests that exercise `main`, `_find_db`, and the registered tools
- [general] `mcp-stdio-transport` — How `FastMCP.run()` speaks JSON-RPC over stdin/stdout and why stdout must remain clean
- [general] `ftl-reasons-entry-points` — How the `reasons-mcp` console script is declared and dispatches to this `main`

## Beliefs

- `main-resolves-db-eagerly` — `main` assigns `_db` via `_find_db()` before calling `mcp.run()`, so all tool calls use the same path without re-discovery.
- `main-requires-mcp-package` — Importing `reasons.mcp_server` fails with `ImportError` if the `mcp` package is unavailable, before `main` can run.
- `main-finds-db-without-validating-existence` — `_find_db()` may return `api.DEFAULT_DB` for a path that does not exist; `main` does not verify the file is present or readable.
- `main-blocks-on-run` — `mcp.run()` is a blocking stdio transport loop, so `main` normally never returns during server operation.
- `main-shares-db-with-get-db` — `main` and `_get_db` intentionally share the module global `_db`, and `_get_db`'s `None`-check makes the eager assignment in `main` idempotent.
