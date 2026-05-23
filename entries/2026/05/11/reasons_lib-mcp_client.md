# File: reasons_lib/mcp_client.py

**Date:** 2026-05-11
**Time:** 13:06

Good, I have the full picture. Here's the explanation:

---

## `reasons_lib/mcp_client.py` — Synchronous Bridge to MCP Servers

### 1. Purpose

This module is the **MCP (Model Context Protocol) client adapter** for the ftl-reasons project. Its sole job is to let the synchronous Python codebase — specifically `ask.py`'s tool loop — talk to external MCP servers over stdio without dealing with async code directly.

MCP servers expose tools (like search, query, or analysis capabilities) over a standardized protocol. This module wraps the async MCP SDK into a synchronous `McpBridge` class so that `ask.py` can call `bridge.call_tool(name, args)` as if it were a normal blocking function call.

### 2. Key Components

**`_mcp_available` (bool)**: Module-level flag set at import time. The `mcp` package is an optional dependency (installed via `pip install "ftl-reasons[mcp]"`), so the import is wrapped in try/except. This lets the rest of the codebase import `mcp_client` unconditionally — it only fails at runtime if you actually try to construct an `McpBridge`.

**`_require_mcp()`**: Guard function called in `McpBridge.__init__`. Raises `ImportError` with install instructions if the `mcp` package isn't available. Centralizes the error message.

**`McpBridge`**: The core class. One instance per MCP server connection. It:

- **`__init__(command)`** — Takes a shell command string (e.g., `"npx @modelcontextprotocol/server-filesystem /tmp"`). Initializes all state to empty/None and creates a `threading.Event` for synchronization.
- **`connect()`** — Spins up a background thread running a dedicated asyncio event loop, then blocks the calling thread on `self._ready` (up to 30s) until the MCP session is initialized.
- **`list_tools()`** — Returns the tool catalog (list of dicts with `name`, `description`, `input_schema`) that was populated during `connect()`.
- **`get_instructions()`** — Returns server-provided instructions string (from `session.initialize()`), used by `ask.py` to inject server context into the LLM prompt.
- **`call_tool(name, arguments)`** — Synchronous tool invocation. Submits the async call via `run_coroutine_threadsafe` onto the background loop, blocks for up to 60s. Returns the result as a joined string of all content parts.
- **`close()`** — Signals the background loop to shut down via `_shutdown.set()`, joins the thread (5s timeout), then closes the loop.

### 3. Patterns

**Background event loop thread**: The central pattern. Async MCP sessions need a running event loop to keep the stdio transport alive. Rather than making the entire call chain async, the module runs `asyncio.new_event_loop()` on a daemon thread. The main thread communicates via `run_coroutine_threadsafe` (for tool calls) and `threading.Event` (for lifecycle coordination). This is a standard pattern for bridging sync/async boundaries in Python.

**Deferred lifecycle with shutdown event**: `_session_lifecycle` runs the full MCP session inside nested `async with` blocks (stdio transport + client session). After initialization, it parks on `await self._shutdown.wait()`. This keeps the session alive indefinitely — tool calls can be submitted from the main thread — until `close()` signals the event. When the event fires, the `async with` blocks unwind and the session closes cleanly.

**Context manager protocol**: `__enter__`/`__exit__` let you use `with McpBridge(cmd) as bridge:` for automatic connect/close lifecycle.

**Lazy import guard**: The `mcp` package is optional. The try/except + `_require_mcp()` pattern delays the failure to construction time rather than import time, so `cli.py` can import this module unconditionally and only fail if the user actually invokes an MCP-related command.

### 4. Dependencies

**Imports**:
- `asyncio` — Event loop management, cross-thread coroutine submission
- `shlex` — Splits the command string into argv for `StdioServerParameters`
- `threading` — Background thread + `Event` for sync coordination
- `mcp` (optional) — `ClientSession`, `StdioServerParameters`, `stdio_client` from the MCP SDK

**Depended on by**:
- `reasons_lib/cli.py` — Imports `McpBridge` inside the `cmd_ask` handler (deferred import pattern), constructs one per `--mcp-server` argument
- `reasons_lib/ask.py` — Receives connected `McpBridge` instances via the `mcp_servers` parameter; calls `list_tools()`, `get_instructions()`, and `call_tool()` within the tool loop

### 5. Flow

```
User runs: reasons ask --mcp-server "npx some-server" "my question"

cli.py:cmd_ask()
  ├─ from reasons_lib.mcp_client import McpBridge
  ├─ for each --mcp-server arg:
  │    ├─ bridge = McpBridge(command)
  │    └─ bridge.connect()
  │         ├─ Spawns daemon thread running _run_loop()
  │         │    └─ _session_lifecycle():
  │         │         ├─ shlex.split(command) → StdioServerParameters
  │         │         ├─ stdio_client() → (read, write) streams
  │         │         ├─ ClientSession(read, write)
  │         │         ├─ session.initialize() → captures instructions
  │         │         ├─ session.list_tools() → populates self._tools
  │         │         ├─ self._ready.set()  ← unblocks main thread
  │         │         └─ await self._shutdown.wait()  ← parks here
  │         └─ _ready.wait(30s) ← main thread unblocks
  │
  ├─ ask(question, mcp_servers=[bridge, ...])
  │    ├─ _build_tools_section(bridges) → tool catalog for prompt
  │    ├─ _build_mcp_instructions(bridges) → server instructions
  │    └─ tool loop:
  │         ├─ LLM emits {"tool": "mcp_tool_name", ...}
  │         └─ bridge.call_tool(name, args)
  │              ├─ run_coroutine_threadsafe(_call_tool_async, loop)
  │              ├─ _call_tool_async: session.call_tool(name, args)
  │              └─ returns joined text of content parts
  │
  └─ bridge.close()
       ├─ _shutdown.set()  ← unparks background loop
       ├─ thread.join(5s)
       └─ loop.close()
```

### 6. Invariants

- **`_require_mcp()` is called before any MCP SDK usage**: The `__init__` constructor is the only entry point, and it calls `_require_mcp()` first. No MCP SDK types are referenced at module level outside the guarded import.
- **`connect()` blocks until the session is fully initialized or fails**: The `_ready` event is only set after `session.initialize()` and `session.list_tools()` complete (or after an exception). The main thread cannot proceed with a half-initialized bridge.
- **`_tools` and `_instructions` are populated exactly once during connect**: There's no refresh mechanism. The tool catalog is a snapshot from initialization time.
- **`call_tool` has a 60-second hard timeout**: If the MCP server hangs, the calling thread unblocks after 60s with a `TimeoutError` (from `future.result(timeout=60)`).
- **The background thread is a daemon**: If the main process exits without calling `close()`, the thread dies with the process. No zombie threads, but also no clean session teardown.

### 7. Error Handling

- **MCP not installed**: `_require_mcp()` raises `ImportError` with install instructions. Clean, early failure.
- **Server doesn't respond within 30s**: `connect()` raises `TimeoutError`. The `_ready.wait(timeout=30)` returns `False`, which triggers the explicit raise.
- **Session initialization fails**: The `except Exception as e` block in `_session_lifecycle` captures the error into `self._error` and sets `_ready`. Back in `connect()`, the error is re-raised after `_ready` fires. This ensures the main thread always sees the failure, even though it happened on a different thread.
- **Tool call fails**: Not caught here — the `future.result()` in `call_tool` will propagate whatever exception the async call raised. The caller (`ask.py`) catches these with a generic `except Exception` and injects the error text into the tool history for the LLM to see.
- **`close()` is tolerant of partial initialization**: Guards on `self._loop and self._shutdown` / `self._thread` before acting, so calling `close()` on a bridge that never connected doesn't crash.

---

## Topics to Explore

- [file] `reasons_lib/ask.py` — The consumer of `McpBridge`; its tool loop dispatches to `bridge.call_tool()` and uses `list_tools()`/`get_instructions()` to build prompts
- [file] `reasons_lib/cli.py` — Where `--mcp-server` arguments are parsed and `McpBridge` instances are constructed and connected
- [file] `tests/test_ask_mcp.py` — Test suite showing how MCP integration is exercised (likely with mock servers)
- [general] `mcp-sdk-stdio-transport` — The `stdio_client` context manager from the MCP SDK that manages the subprocess lifecycle for the child server process
- [function] `reasons_lib/ask.py:_build_tools_section` — How the bridge's tool catalog is formatted into the LLM prompt, bridging MCP tool schemas to the text-based tool-calling protocol

## Beliefs

- `mcp-bridge-runs-dedicated-event-loop-thread` — Each `McpBridge` instance runs its own asyncio event loop on a daemon thread; the session stays alive until `close()` signals the shutdown event
- `mcp-bridge-connect-blocks-up-to-30s` — `connect()` blocks the calling thread for up to 30 seconds waiting for session initialization, then raises `TimeoutError`
- `mcp-bridge-call-tool-timeout-60s` — `call_tool()` blocks for up to 60 seconds per invocation via `future.result(timeout=60)`
- `mcp-bridge-tools-snapshot-at-connect` — The tool catalog and server instructions are populated once during `connect()` and never refreshed
- `mcp-is-optional-dependency` — The `mcp` package is guarded by try/except at import time; `_require_mcp()` defers the `ImportError` to construction time so the module can be imported unconditionally

