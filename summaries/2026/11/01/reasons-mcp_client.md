---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/mcp_client.py

# `reasons/mcp_client.py`

## Purpose

This file is a **synchronous-to-asynchronous bridge** that lets the rest of the `reasons` codebase (which is entirely synchronous) dispatch tool calls to an external MCP (Model Context Protocol) server. The MCP SDK (`mcp`) is asyncio-native, but `ask.py` and the CLI operate in plain synchronous Python. `McpBridge` hides that impedance mismatch behind a simple synchronous API: `connect()`, `list_tools()`, `call_tool()`, `close()`.

Its single responsibility: **keep one persistent MCP session alive on a background event-loop thread and expose it via blocking method calls.**

## Key Components

### `_require_mcp()` (line 22)
A guard that raises `ImportError` with a pip-install hint if the optional `mcp` package is missing. Called once in `McpBridge.__init__`, so the failure is immediate and explicit rather than deferred.

### `McpBridge` class (line 30)
The sole public class. Its contract:

| Member | Contract |
|--------|----------|
| `__init__(command: str)` | Stores the shell command (e.g. `"npx @mcp/server-fs /path"`). Does **not** connect. |
| `connect()` | Spawns a daemon thread, runs the MCP handshake, blocks until ready (30 s) or raises. |
| `list_tools() → list[dict]` | Returns the cached tool descriptors populated during `connect()`. Shape: `{"name", "description", "input_schema"}`. |
| `get_instructions() → str` | Returns the server's free-text instructions (may be `""`). |
| `call_tool(name, arguments) → str` | Blocking call; returns the concatenated text content of the tool result. 60 s timeout. |
| `close()` | Signals shutdown, joins the thread (5 s), closes the loop. Idempotent (null-guards). |
| `__enter__ / __exit__` | Context-manager sugar: `with McpBridge(cmd) as bridge:` auto-connects and closes. |

### Internal state
- `self._loop` — the dedicated `asyncio` event loop living on the background thread.
- `self._thread` — the daemon thread running that loop.
- `self._session` — the `mcp.ClientSession` handle (async object, only touched inside the loop).
- `self._shutdown` — an `asyncio.Event` that unblocks `_session_lifecycle` so the loop can exit.
- `self._ready` — a `threading.Event` used for the cross-thread handshake.
- `self._error` — set on exception during lifecycle; re-raised by `connect()`.

### `_session_lifecycle()` (line 72)
The async "main body" that runs for the lifetime of the bridge:

1. `shlex.split(self.command)` → argv list.
2. `stdio_client(StdioServerParameters(...))` → spawns the server as a child process over stdio.
3. `ClientSession(read, write)` → the MCP protocol session.
4. `session.initialize()` → protocol handshake; captures optional `instructions`.
5. `session.list_tools()` → populates `self._tools`.
6. Sets `self._ready` (signals the calling thread that setup is done).
7. `await self._shutdown.wait()` → parks the coroutine until `close()` fires the event.

### `_call_tool_async()` (line 119)
Thin wrapper: calls `self._session.call_tool(name, arguments)`, then flattens `result.content` (a list of content blocks) into a single `\n`-joined string. Non-text blocks are stringified via `str()`.

## Patterns

- **Bridge / Adapter**: The classic sync→async bridge. The caller never touches `asyncio`; all futures are resolved inside `call_tool` via `future.result(timeout=60)`.
- **Background event-loop thread**: A single daemon thread owns the loop for the entire session lifetime. This keeps the MCP session (and its child process) alive across multiple `call_tool` invocations without the caller needing to manage async context.
- **Handshake with `threading.Event`**: `_ready` + `_error` form a simple cross-thread "I'm up" or "I failed" signal. No locks, no queues.
- **Context manager**: `__enter__`/`__exit__` make the resource lifecycle explicit at the call-site: `with McpBridge("npx ...") as bridge:`.
- **Optional dependency with import-time guard**: The `try/except ImportError` at module top sets `_mcp_available`; `_require_mcp()` defers the hard failure to construction time, so importing `reasons.cli` doesn't crash if `mcp` isn't installed.
- **Daemon thread**: The MCP thread won't block process exit. If the caller forgets `close()`, the thread dies with the process (the child MCP server is a stdio child and gets killed by the OS).

## Dependencies

### Imports (this file → world)
- `asyncio` — event loop, `run_coroutine_threadsafe`, `Event`
- `shlex` — splitting the server command string
- `threading` — background thread, `threading.Event`
- `mcp` (optional) — `ClientSession`, `StdioServerParameters`, `stdio_client`

### Imported by (world → this file)
- `reasons/cli.py` — the CLI entry point that wires `--mcp` flags into a `McpBridge` instance and passes it to the ask/review pipeline.
- `reasons_lib/mcp_client.py` — the published-library mirror (identical content, different package path for the `pip install ftl-reasons` distribution).

## Flow

```
Caller (sync)                        Background thread (async)
─────────────                        ─────────────────────────
bridge = McpBridge(cmd)
bridge.connect()
  ├─ new event loop
  ├─ start daemon thread ─────────►  _run_loop()
  │                                  └─ loop.run_until_complete(
  │                                     _session_lifecycle())
  │                                       ├─ shlex.split(cmd)
  │                                       ├─ spawn stdio child proc
  │                                       ├─ ClientSession init
  │                                       ├─ list_tools() → _tools
  │                                       ├─ _ready.set() ◄──┐
  │                                       └─ await _shutdown.wait()
  ├─ _ready.wait(30) ◄─────────────────────┘
  └─ return  (or raise TimeoutError / captured error)

bridge.call_tool("search", {"q":"…"})
  ├─ run_coroutine_threadsafe(
  │     _call_tool_async(...), loop) ──►  await session.call_tool(...)
  │                                       → flatten content → str
  └─ future.result(60)  ◄────────────────┘
     → return "result text"

bridge.close()
  ├─ loop.call_soon_threadsafe(_shutdown.set) ──►  _session_lifecycle unblocks
  │                                                  → async-with exits (kills child)
  ├─ thread.join(5)
  └─ loop.close()
```

The critical detail: **all async work is serialised onto one event loop**. `run_coroutine_threadsafe` is the only cross-thread entry point, so there's no contention on the session object.

## Invariants

1. **Single-session guarantee**: Exactly one `ClientSession` exists at a time. The bridge is not meant to multiplex multiple servers.
2. **`connect()` before any call**: `list_tools()`, `call_tool()`, and `get_instructions()` assume `_ready` has been set. Calling them before `connect()` returns `[]`, `""`, or raises `AttributeError` on `self._loop`.
3. **Command is a single shell string**: `shlex.split` expects one command. You can't pass a list of args directly—you must pass a string like `"npx @mcp/server-fs --root /data"`.
4. **Thread affinity**: `self._session` is only valid inside `self._loop`. All access goes through `run_coroutine_threadsafe` or runs within the loop itself. Accessing `_session` from the main thread would be a runtime error.
5. **60-second tool-call budget**: Any tool that takes longer than 60 s will raise `concurrent.futures.TimeoutError`. There is no retry logic.

## Error Handling

| Scenario | Behaviour |
|----------|-----------|
| `mcp` not installed | `ImportError` at `McpBridge.__init__` (via `_require_mcp`). Import of the module itself succeeds. |
| Server doesn't start / handshake fails within 30 s | `TimeoutError` from `connect()`. |
| Exception during `_session_lifecycle` (e.g. server crashes immediately) | Captured into `self._error`, `_ready` is set, `connect()` re-raises the original exception. |
| Server dies mid-session (after `connect`) | Next `call_tool` → `future.result()` raises the underlying `ConnectionError` / `EOFError` wrapped in `ConcurrentFuture`. Not caught here; propagates to caller. |
| Tool call exceeds 60 s | `concurrent.futures.TimeoutError`. The coroutine is **not** cancelled (Python limitation); it may still complete in the background. |
| `close()` called twice | Null-guards on `self._loop` and `self._thread` make the second call a no-op. |
| `close()` not called | Daemon thread + stdio child are killed by the OS on process exit. No explicit cleanup guarantee. |

Notably, there is **no retry logic, no reconnection, and no health-check**. The bridge is a one-shot session. If the server dies, the caller must create a new `McpBridge` and `connect()` again.

## Topics to Explore

- [file] `reasons/ask.py` — The primary consumer that receives a `McpBridge` and dispatches tool calls; understanding how it interprets `list_tools()` output and handles the returned strings will clarify the bridge's contract.
- [file] `reasons/cli.py` — Shows how the `--mcp` CLI flag is parsed, the command string assembled, and the bridge injected into the pipeline.
- [function] `reasons/mcp_client.py:_session_lifecycle` — The 50-line async function that owns the entire connection lifecycle; any protocol changes (e.g. adding `list_prompts`, sampling) land here.
- [file] `tests/test_ask_mcp.py` — Exercises the bridge end-to-end with a mock or real MCP server; reveals expected error shapes and the 30 s / 60 s timeout behaviour.
- [general] `mcp-stdio-server-selection` — How the project decides which MCP server to spawn (env vars, config files, CLI flags) and what tool-name conventions the downstream ask/review code expects.

## Beliefs

- `mcp-bridge-requires-mcp-sdk` — `McpBridge.__init__` raises `ImportError` if the `mcp` package is not installed; the module is importable without it, but the class is unusable.
- `mcp-bridge-maps-to-sync-api` — `McpBridge.call_tool(name, arguments)` is the sole synchronous entry point for tool dispatch; it blocks for at most 60 s and returns a flat `str`, never a structured `CallToolResult`.
- `mcp-bridge-contains-single-session` — Exactly one `ClientSession` and one stdio child process exist per `McpBridge` instance; there is no multiplexing, pooling, or reconnection logic.
- `mcp-bridge-prevents-multithreaded-session-access` — All session mutations are serialised through a single `asyncio` event loop on one daemon thread; concurrent `call_tool` invocations from multiple OS threads are safe only because they funnel through `run_coroutine_threadsafe` onto that loop.
- `mcp-bridge-lifecycle-is-ctx-manager-bound` — The intended usage is `with McpBridge(cmd) as bridge:`; calling `connect()` and forgetting `close()` leaks the stdio child process until process exit (the daemon thread provides no safety net for the child).
