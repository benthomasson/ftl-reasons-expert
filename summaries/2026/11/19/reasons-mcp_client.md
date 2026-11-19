---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/mcp_client.py

# `reasons/mcp_client.py`

## Purpose

This module provides a synchronous adapter around the async MCP Python SDK. It launches an MCP server over stdio, keeps the async `ClientSession` alive in a background event-loop thread, and exposes blocking methods for tool discovery and tool invocation.

Its responsibility is transport/lifecycle only: it does not validate reasoning data or implement tool logic. It lets synchronous calling code, such as the CLI/ask flow, treat any MCP server as a data-source adapter.

The module is import-safe when `mcp` is not installed; attempting to construct `McpBridge` raises a clear `ImportError`.

## Key Components

### Optional MCP import guard

```python
try:
    from mcp import ClientSession, StdioServerParameters
    from mcp.client.stdio import stdio_client
    _mcp_available = True
except ImportError:
    _mcp_available = False
```

- `_mcp_available` records whether the optional `mcp>=1.0` dependency was importable.
- `_require_mcp()` raises `ImportError` with an install hint if `McpBridge` is used without the dependency.

### `McpBridge`

Synchronous bridge to an MCP server over stdio.

Constructor:

```python
McpBridge(command)
```

- `command` is a shell-like command string, later parsed with `shlex.split`.
- Calls `_require_mcp()` immediately, so the bridge cannot be constructed without `mcp`.
- Initializes cached state: `_tools`, `_instructions`, `_loop`, `_thread`, `_session`, `_shutdown`, `_ready`, `_error`.

Fields:

| Field | Meaning |
|---|---|
| `_tools` | Cached list of tool metadata dicts discovered during `connect()`. |
| `_instructions` | Cached instructions from `session.initialize()`, or `""`. |
| `_loop` | Background asyncio event loop. |
| `_thread` | Daemon thread running `_loop`. |
| `_session` | Live MCP `ClientSession`, valid while the async context managers are open. |
| `_shutdown` | `asyncio.Event` used to keep the session lifecycle alive until `close()`. |
| `_ready` | `threading.Event` signaling that startup completed or failed. |
| `_error` | Startup exception captured by `_session_lifecycle()` and re-raised by `connect()`. |

### `connect()`

- Creates a new asyncio event loop and starts `_run_loop` in a daemon thread.
- Waits up to 30 seconds on `_ready`.
- Raises `TimeoutError` if the server does not signal readiness in time.
- Re-raises `_error` if initialization/list-tools failed.
- Must be called before `call_tool()`.

### `_run_loop()` / `_session_lifecycle()`

`_run_loop()` sets the thread-local event loop and runs `_session_lifecycle()` to completion.

`_session_lifecycle()`:

1. Parses `self.command` via `shlex.split`.
2. Builds `StdioServerParameters(command=parts[0], args=parts[1:])`.
3. Opens `stdio_client(...)`.
4. Opens `ClientSession(read, write)`.
5. Calls `session.initialize()` and caches `instructions` if present.
6. Calls `session.list_tools()` and caches each tool as:
   ```python
   {
       "name": t.name,
       "description": t.description or "",
       "input_schema": t.inputSchema,
   }
   ```
7. Creates `_shutdown = asyncio.Event()`.
8. Sets `_ready`.
9. Awaits `_shutdown.wait()` to keep the session alive until `close()`.

If initialization or tool listing raises inside the `try`, the exception is stored in `_error`, `_ready` is set, and `connect()` re-raises it.

### `list_tools()` and `get_instructions()`

- `list_tools()` returns the cached `_tools` list. It does not re-query the server after startup.
- `get_instructions()` returns cached server instructions, or `""`.

Before `connect()`, these return the initial empty values.

### `call_tool(name, arguments)`

- Submits `_call_tool_async(name, arguments)` to the background loop using `asyncio.run_coroutine_threadsafe`.
- Blocks on the returned future with a 60-second timeout.
- Returns a string produced by joining MCP result content.

`_call_tool_async()`:

```python
result = await self._session.call_tool(name, arguments)
parts = []
for content in result.content:
    if hasattr(content, "text"):
        parts.append(content.text)
    else:
        parts.append(str(content))
return "\n".join(parts)
```

So text content is concatenated with newlines; non-text content is stringified.

### `close()` and context manager support

- `close()` signals `_shutdown` through `loop.call_soon_threadsafe(...)`.
- Joins the background thread with a 5-second timeout.
- Closes and clears the event loop.
- `__enter__()` calls `connect()` and returns `self`.
- `__exit__()` calls `close()`.

This supports:

```python
with McpBridge("python server.py") as bridge:
    tools = bridge.list_tools()
    result = bridge.call_tool("some_tool", {"x": 1})
```

## Patterns

- **Adapter / bridge pattern**: converts the async MCP SDK API into a synchronous API.
- **Background event loop thread**: keeps async session state alive across multiple synchronous calls.
- **Event-based synchronization**:
  - `threading.Event` for startup readiness.
  - `asyncio.Event` for shutdown coordination.
- **Cross-thread async submission**:
  - `asyncio.run_coroutine_threadsafe` for tool calls.
  - `loop.call_soon_threadsafe` for shutdown signaling.
- **Context manager lifecycle**: ensures shutdown after successful connection.
- **Lazy optional dependency handling**: module imports without `mcp`; usage fails with a targeted error.

## Dependencies

### Imports

- Standard library: `asyncio`, `shlex`, `threading`.
- Optional third-party: `mcp>=1.0`.
  - `ClientSession`
  - `StdioServerParameters`
  - `stdio_client`

### Imported by

According to the repository context:

- `reasons/cli.py`
- `reasons_lib/mcp_client.py`

The docstring describes its intended use by the ask/tool-dispatch flow, while the CLI is the direct integration point shown in the repository.

## Flow

1. Caller constructs `McpBridge(command)`.
2. `__init__` verifies `mcp` is installed and initializes synchronization state.
3. `connect()` starts a daemon thread running a dedicated asyncio loop.
4. `_session_lifecycle()` parses the command, launches the MCP server over stdio, opens a `ClientSession`, initializes it, fetches instructions, and lists tools.
5. `_ready` is set, unblocking `connect()`.
6. The lifecycle coroutine waits on `_shutdown`; this keeps the `ClientSession` context open.
7. Caller calls `list_tools()`, `get_instructions()`, or `call_tool(...)`.
8. `call_tool()` submits the coroutine to the background loop and blocks for the string result.
9. `close()` signals `_shutdown`, causing the async context managers to exit, then joins the thread and closes the loop.

## Invariants

- `McpBridge` requires the `mcp` package to be installed.
- `connect()` must complete successfully before `call_tool()` is useful.
- `_session` is only valid while `_session_lifecycle()` is inside the `ClientSession` context.
- The session is kept alive by `await self._shutdown.wait()` inside the lifecycle coroutine.
- `command` is parsed with `shlex.split`; the first token must be the executable and remaining tokens are arguments. No shell is used.
- Tool metadata is fetched once at connect time and cached; later server-side tool changes are not reflected.
- `call_tool()` does not validate `arguments` against the tool’s `input_schema`; validation is the server/SDK’s responsibility.
- `close()` relies on `_shutdown` having been created. If startup fails before `_shutdown` is assigned, `close()` cannot signal the async lifecycle.

## Error Handling

- Missing `mcp` dependency:
  - `__init__` calls `_require_mcp()` and raises `ImportError` with install instructions.
- Startup timeout:
  - If `_ready` is not set within 30 seconds, `connect()` raises:
    ```python
    TimeoutError(f"MCP server '{self.command}' did not respond within 30s")
    ```
- Initialization/tool-listing failure:
  - Exceptions inside the `try` in `_session_lifecycle()` are stored in `_error`, `_ready` is set, and `connect()` re-raises the original exception.
- Pre-`try` startup failure:
  - Errors from `shlex.split`, `parts[0]`, `StdioServerParameters`, `stdio_client`, or `ClientSession.__aenter__` occur before the `try` block. These are not written to `_error`, so a caller may see a 30-second timeout instead of the underlying exception.
- Tool-call failure:
  - Exceptions from `session.call_tool()` propagate through the future returned by `run_coroutine_threadsafe`.
  - `future.result(timeout=60)` raises a timeout exception if the tool call does not finish within 60 seconds. The coroutine is not explicitly cancelled.
- Shutdown:
  - `close()` joins the thread with a 5-second timeout and closes the loop. It does not otherwise catch shutdown exceptions.

## Topics to Explore

- [file] `reasons/cli.py` — Shows how `McpBridge` is constructed, configured, and wired into the CLI/ask flow.
- [function] `reasons/mcp_client.py:McpBridge._session_lifecycle` — The critical async lifecycle: command parsing, server launch, initialization, tool caching, shutdown waiting, and error capture.
- [file] `reasons_lib/mcp_client.py` — Mirrored module path exists in the repository; compare it with `reasons/mcp_client.py` to keep implementations in sync.
- [file] `reasons/mcp_server.py` — Server-side MCP implementation whose tools and instructions this client consumes.
- [general] `mcp-sdk-tool-result-semantics` — Understand `CallToolResult.content`, error flags, and how `ClientSession.call_tool` raises or returns tool errors.

## Beliefs

- `mcpbridge-requires-mcp-package` — Constructing `McpBridge` raises `ImportError` when the optional `mcp` package is unavailable.
- `mcpbridge-keeps-session-open-until-shutdown-event` — `_session_lifecycle` awaits `_shutdown.wait()` inside the `stdio_client` and `ClientSession` context managers to keep the MCP session alive.
- `mcpbridge-caches-tools-at-connect` — Tool metadata is fetched once during `connect()` and returned from `list_tools()` without re-querying the server.
- `mcpbridge-call-tool-flattens-content` — `call_tool()` returns newline-joined text or stringified content from the MCP `CallToolResult`.
- `mcpbridge-close-uses-thread-safe-signaling` — `close()` sets the asyncio shutdown event through `loop.call_soon_threadsafe`.
