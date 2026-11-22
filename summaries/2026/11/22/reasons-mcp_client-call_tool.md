---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: call_tool in reasons/mcp_client.py

## Purpose

`call_tool` is the synchronous entry point for invoking a tool on a connected MCP server. It exists so callers like `ask.py` can stay synchronous while `McpBridge` keeps a persistent async MCP session alive on a background event-loop thread.

It schedules `_call_tool_async(name, arguments)` onto `self._loop`, then blocks the calling thread until the resulting future completes or 60 seconds elapse.

## Contract

Preconditions:

- `connect()` must have completed successfully.
- `self._loop` must exist and be running.
- `self._session` must be initialized and usable.
- The MCP dependency must be installed.
- `name` should be a tool name advertised by `list_tools()`.
- `arguments` should match the tool’s `input_schema`.

Postconditions on success:

- The MCP tool has been invoked through `self._session.call_tool(...)`.
- The caller receives a `str` produced by flattening `result.content`.
- No bridge fields are intentionally mutated by `call_tool` itself.

Important invariants / assumptions not enforced by the type system:

- `call_tool` assumes `self._loop` is the same loop that owns `self._session`.
- It assumes `self._session` is valid; it does not check for `None` or closed state.
- It assumes the caller is not running on the background event-loop thread; otherwise blocking on `future.result()` could deadlock.
- It assumes the MCP SDK/session supports whatever concurrency pattern callers use.
- It assumes text content values are join-safe strings when `content.text` exists.

## Parameters

### `name`

Expected type: `str`.

The exact MCP tool name, typically from `McpBridge.list_tools()`. No validation is performed here. An unknown or invalid name is passed to the MCP session and any resulting error propagates through the future.

### `arguments`

Expected type: generally `dict`.

Passed directly to `self._session.call_tool(name, arguments)`. No schema validation is performed here. Invalid arguments are rejected by the MCP server/SDK and surface as exceptions, unless the server reports the error as normal tool content.

Edge cases:

- `{}` is valid for tools with no required arguments.
- `None` may be passed syntactically, but the MCP SDK/server may reject it.
- Non-JSON-serializable values may fail during MCP transport/serialization.

## Return Value

Returns a `str` on success.

The string is built in `_call_tool_async`:

```python
parts = []
for content in result.content:
    if hasattr(content, "text"):
        parts.append(content.text)
    else:
        parts.append(str(content))
return "\n".join(parts)
```

So:

- Multiple content items are joined with newline characters.
- Items with a `text` attribute contribute `content.text`.
- Other content items are converted via `str(content)`.
- Empty `result.content` returns `""`.
- The raw MCP `CallToolResult` is not returned.
- Metadata such as `isError`, structured content, and content type information are discarded.

The caller must handle exceptions and timeouts; this method does not return error sentinel values.

## Algorithm

1. Create the coroutine object:

   ```python
   self._call_tool_async(name, arguments)
   ```

2. Schedule it on the persistent background event loop using:

   ```python
   asyncio.run_coroutine_threadsafe(..., self._loop)
   ```

   This returns a `concurrent.futures.Future`.

3. Block the calling thread:

   ```python
   future.result(timeout=60)
   ```

4. On the background event loop, `_call_tool_async` runs:

   ```python
   result = await self._session.call_tool(name, arguments)
   ```

5. It iterates over `result.content`, collecting text or stringified content.

6. It joins the collected parts with `"\n"` and returns the resulting string.

7. `future.result()` either returns that string or re-raises the coroutine’s exception.

## Side Effects

- Schedules work onto the background asyncio loop.
- Performs MCP I/O over stdio to the server subprocess.
- Blocks the calling thread for up to 60 seconds.
- May leave the underlying MCP call running after a timeout, because `future.result(timeout=60)` does not cancel the future/coroutine.
- If scheduling itself fails, the created coroutine may never be awaited, potentially producing a Python “coroutine was never awaited” warning.
- It does not explicitly mutate `McpBridge` fields, but the MCP session/server may change state as a result of the tool call.

## Error Handling

`call_tool` has no `try/except`. Errors propagate to the caller.

Possible failures include:

- `AttributeError` or similar if `connect()` was never called and `self._loop` is `None`.
- `RuntimeError` if the event loop is already closed.
- Hanging until timeout if `self._loop` exists but is not running.
- `concurrent.futures.TimeoutError` after 60 seconds. In newer Python versions this is aliased to built-in `TimeoutError`.
- Exceptions raised by `_call_tool_async`, including MCP transport/session errors, invalid tool names, invalid arguments, or formatting errors such as joining non-string `content.text`.
- No explicit handling of MCP `CallToolResult.isError`. If the server reports a tool error through `isError=True` but still returns content, this wrapper will return that content as a normal string unless the SDK itself raises.

Timeout semantics are important: a timeout raises to the caller, but the scheduled coroutine is not cancelled by this code.

## Usage Patterns

Typical usage:

```python
with McpBridge("python server.py") as bridge:
    tools = bridge.list_tools()
    result = bridge.call_tool("some_tool", {"arg": 1})
```

Caller obligations:

- Call `connect()` first, usually via the context manager.
- Do not call after `close()`.
- Do not call from the background event-loop thread.
- Catch `TimeoutError` / MCP exceptions as appropriate.
- Treat the return value as flattened text, not as a structured MCP result.
- Serialize calls if the MCP session is not safe for concurrent tool calls.

## Dependencies

Internal:

- `McpBridge._call_tool_async`
- `McpBridge.connect` / `McpBridge._session_lifecycle` indirectly, because they create and own `self._loop` and `self._session`
- `McpBridge.list_tools` indirectly, because callers usually use it to discover valid tool names

External:

- `asyncio.run_coroutine_threadsafe`
- `concurrent.futures.Future` behavior via `future.result`
- MCP SDK: `ClientSession.call_tool`
- MCP stdio transport/session lifecycle from the surrounding bridge

## Topics to Explore

- [file] `tests/test_ask_mcp.py` — Shows how the MCP bridge is exercised and what behavior the tests expect.
- [function] `reasons/mcp_client.py:_call_tool_async` — Formats MCP `CallToolResult` content and ignores structured/error metadata.
- [function] `reasons/mcp_client.py:close` — Shutdown ordering and races with in-flight `call_tool` calls.
- [general] `mcp-calltoolresult-iserror` — How MCP tool-level errors are represented and why this wrapper may return them as text.
- [general] `mcp-client-session-concurrency` — Whether `ClientSession` supports concurrent tool calls from multiple threads.

## Beliefs

- `call-tool-requires-connected-event-loop` — `call_tool` assumes `connect()` already created a running `_loop`; it does not check or initialize it.
- `call-tool-blocks-caller-for-up-to-60-seconds` — `future.result(timeout=60)` blocks the calling thread with a hardcoded 60-second timeout.
- `call-tool-timeout-does-not-cancel-operation` — A `future.result` timeout raises to the caller but does not cancel the scheduled coroutine, so the MCP call may continue.
- `call-tool-returns-flattened-text-content` — On success it returns `result.content` texts joined by newlines, converting non-text items with `str()`.
- `call-tool-ignores-calltoolresult-iserror` — The method never checks `result.isError`, so server-reported tool errors are returned as normal text unless the SDK raises.
