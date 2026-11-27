---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Function: close in reasons/mcp_client.py

## Purpose

`McpBridge.close` tears down the persistent background asyncio event loop and the MCP stdio session created by `connect`.

It exists because `McpBridge` keeps an asyncio loop running in a daemon thread so synchronous callers can invoke async MCP operations repeatedly. `close` must:

1. Ask the async lifecycle coroutine to exit gracefully.
2. Wait for the background thread to finish.
3. Close the event loop.

Without this, the MCP session, stdio transport, event loop, and background thread could leak or remain alive after the bridge is no longer needed.

## Contract

### Preconditions

- No strict preconditions are enforced. Calling `close` before `connect` is effectively a no-op.
- Intended usage: call after a successful `connect`, or via `__exit__` after using the object as a context manager.
- The caller should not call `close` from the background event-loop thread. `thread.join()` would raise `RuntimeError` when joining the current thread.
- The caller should not use the bridge after `close`.
- The caller should not concurrently call `call_tool` while `close` is running. There is no synchronization protecting that race.

### Postconditions

- If `_loop` and `_shutdown` are both non-`None`, the shutdown event is scheduled to be set on the event loop thread.
- If `_thread` is non-`None`, `close` attempts to join it for up to 5 seconds.
- `_thread` is set to `None` after `join` returns, regardless of whether the thread is still alive.
- If `_loop` is non-`None`, `close` attempts to close it and then sets `_loop = None`.
- `close` does not clear `_shutdown`, `_session`, `_tools`, `_instructions`, `_ready`, or `_error`.
- `close` returns `None`.

### Invariants / Assumptions Not Enforced by Types

- If `_loop` and `_shutdown` are both non-`None`, the code assumes `_shutdown` belongs to `_loop` and that scheduling `_shutdown.set` on `_loop` is valid.
- The code assumes `_loop` is currently running in `_thread` when shutdown is signaled.
- The code assumes `_thread` is joinable and is not the current thread.
- The code assumes the background thread exits within 5 seconds if `_shutdown` is successfully set. It does not verify that.
- The code assumes `loop.close()` is safe after `join`. It does not check `thread.is_alive()` before closing the loop.
- The object is not safely reusable after `close` because `_ready` remains set. A second `connect` call on the same instance would not wait for the new thread to become ready.

## Parameters

`close(self)` takes no explicit parameters beyond the implicit `McpBridge` instance.

Relevant internal fields:

- `self._loop`: optional `asyncio.AbstractEventLoop`.
- `self._shutdown`: optional `asyncio.Event`, created inside `_session_lifecycle` after MCP initialization succeeds.
- `self._thread`: optional `threading.Thread` running the event loop.

Edge cases:

- `_loop is None` and `_thread is None`: no-op.
- `_shutdown is None`: shutdown is not signaled; the thread is still joined, and the loop is still closed.
- `_loop` already closed: `call_soon_threadsafe` may raise `RuntimeError`.
- `_thread` still alive after 5 seconds: `close` still sets `_thread = None`, then may fail when calling `_loop.close()` on a running loop.
- Called twice: generally safe as a no-op after successful first close, because `_loop` and `_thread` are `None`.

## Return Value

Returns `None`.

The caller gets no indication of whether the thread actually terminated. A join timeout is silent. If teardown fails, `close` raises rather than returning an error object or flag.

## Algorithm

Step by step:

1. If both `_loop` and `_shutdown` exist:
   - Call `self._loop.call_soon_threadsafe(self._shutdown.set)`.
   - This schedules `_shutdown.set()` to run on the event loop thread.
   - When it runs, `_session_lifecycle`’s `await self._shutdown.wait()` completes.
   - The async context managers for `ClientSession` and `stdio_client` then exit, closing the MCP session and stdio transport.

2. If `_thread` exists:
   - Call `self._thread.join(timeout=5)`.
   - This blocks the calling thread for up to five seconds waiting for the background loop thread to finish.
   - The return value of `join` is ignored.
   - `_thread` is then set to `None`.

3. If `_loop` exists:
   - Call `self._loop.close()`.
   - Set `_loop = None` only if `close()` succeeds.

The actual MCP session cleanup is indirect: `close` only signals `_shutdown`; `_session_lifecycle` unwinds the `async with` blocks and performs the real MCP teardown.

## Side Effects

- Schedules an `asyncio.Event.set()` callback on the event loop.
- Causes the background thread to exit if shutdown is successfully signaled.
- Blocks the caller for up to five seconds in `thread.join`.
- Closes the asyncio event loop.
- Mutates `_thread` and `_loop`.
- Leaves `_session`, `_shutdown`, `_tools`, `_instructions`, `_ready`, and `_error` untouched.
- May leave a daemon thread running if shutdown is not signaled or teardown takes longer than five seconds.
- May leak MCP/stdio resources if `_shutdown` is `None` and the async context managers never unwind.

## Error Handling

`close` has no `try`/`except` block. Exceptions propagate to the caller.

Possible exceptions:

- `RuntimeError` from `call_soon_threadsafe` if `_loop` is already closed.
- `RuntimeError` from `thread.join()` if `close` is called from the event-loop thread itself.
- `RuntimeError` from `_loop.close()` if the loop is still running, which can happen if the thread did not exit within five seconds or if shutdown was never signaled.
- Any exception raised during loop shutdown may leave the object partially torn down. For example, `_thread` may already be `None` while `_loop` remains non-`None`.

A join timeout does not raise. It is silently ignored.

## Usage Patterns

Typical usage is through the context manager:

```python
with McpBridge(command) as bridge:
    bridge.call_tool(...)
```

`__exit__` calls `close`.

Manual usage should use `try/finally`:

```python
bridge = McpBridge(command)
bridge.connect()
try:
    bridge.call_tool(...)
finally:
    bridge.close()
```

Callers should:

- Call `close` from the thread that created/owns the bridge, not from the event-loop thread.
- Avoid concurrent `call_tool` and `close`.
- Treat the bridge as unusable after `close`.
- Be prepared for `RuntimeError` if teardown does not complete cleanly.

## Dependencies

- `asyncio`: event loop, event, `call_soon_threadsafe`.
- `threading`: `Thread.join`.
- `McpBridge.connect` / `_session_lifecycle`: responsible for creating `_loop`, `_thread`, and `_shutdown`.
- MCP SDK context managers indirectly: `ClientSession` and `stdio_client` are closed when `_session_lifecycle` unwinds after `_shutdown.set()`.

## Topics to Explore

- [file] `reasons/mcp_client.py` — Full `McpBridge` lifecycle, including `connect`, `_session_lifecycle`, and `close`.
- [function] `reasons/mcp_client.py:_session_lifecycle` — Async context-manager teardown triggered by `close`.
- [function] `reasons/mcp_client.py:connect` — Readiness handling and why `_shutdown` may be `None` during partial startup.
- [general] `asyncio-cross-thread-shutdown` — Semantics of `call_soon_threadsafe`, `thread.join` timeouts, and closing running event loops.
- [general] `mcp-stdio-session-lifecycle` — How MCP `ClientSession` and `stdio_client` release subprocess and stdio resources.

## Beliefs

- `close-signals-shutdown-only-with-loop-and-event` — `close` calls `_shutdown.set` via `call_soon_threadsafe` only when both `_loop` and `_shutdown` are non-`None`.
- `close-join-timeout-is-not-checked` — `close` joins `_thread` with a 5-second timeout but never checks `is_alive` or reports whether the timeout expired.
- `close-clears-thread-even-if-alive` — `close` assigns `_thread = None` after `join`, even if the background thread is still alive.
- `close-does-not-clear-session-state` — `close` leaves `_session`, `_shutdown`, `_tools`, `_instructions`, `_ready`, and `_error` unchanged.
- `close-propagates-teardown-exceptions` — `close` has no `try`/`except`; exceptions from `call_soon_threadsafe`, `join`, or `loop.close` propagate to the caller.
