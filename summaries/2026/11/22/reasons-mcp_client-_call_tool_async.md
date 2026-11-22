---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: _call_tool_async in reasons/mcp_client.py

## Purpose

`_call_tool_async` is the asynchronous implementation of an MCP tool call inside `McpBridge`. The public `McpBridge.call_tool` is synchronous, so it schedules this coroutine on the bridge’s long-lived background event loop with `asyncio.run_coroutine_threadsafe(...)` and blocks on `future.result(timeout=60)`.

Its job is twofold:

1. Call `ClientSession.call_tool(name, arguments)` on the MCP server.
2. Flatten the returned MCP content blocks into one plain string, using each block’s `.text` when available and `str(content)` otherwise.

It exists because the MCP SDK is async and returns structured content objects, while the bridge exposes a synchronous, text-oriented interface.

## Contract

### Preconditions

- The bridge must have been constructed via `McpBridge(command)`, so `_require_mcp()` has already passed.
- `connect()` must have completed successfully.
- `self._session` must be a live, initialized `mcp.ClientSession`.
- This coroutine must run on the same event loop that owns the MCP session, normally `self._loop`. The method itself does not enforce this.
- `name` should identify a tool exposed by the MCP server, typically one returned by `list_tools()`.
- `arguments` should match the tool’s input schema.

### Postconditions

- On success, returns a single `str`.
- Content ordering is preserved from `result.content`.
- If `result.content` is empty, returns `""`.
- The method does not mutate `self` or any bridge state.
- The underlying MCP server/tool may have side effects.

### Invariants

- No exception wrapping or logging occurs here.
- No timeout is applied inside this coroutine.
- The method does not inspect any MCP tool-error flag such as `isError`.
- Content is not parsed semantically; it is either `.text` or `str(content)`.

### Assumptions not enforced by the type system

The function has no type annotations, and Python does not enforce:

- `self._session` is non-`None` and alive.
- `result` has a `content` attribute.
- `result.content` is iterable.
- Any `content.text` value is a string.
- `str(content)` is meaningful for non-text content blocks.
- Multiple concurrent tool calls are safe for the underlying MCP session.
- The tool name and arguments satisfy the server’s schema.

## Parameters

### `name`

The MCP tool name to invoke. Expected to be a string matching a tool advertised by the server. The function performs no validation and passes it directly to `ClientSession.call_tool`.

Edge cases:

- Unknown tool names are left to the MCP SDK/server to reject.
- `None` or non-string values are not checked here.

### `arguments`

The argument payload for the tool. Normally a dictionary matching the tool’s `input_schema`. The function passes it through unchanged and does not validate or mutate it.

Edge cases:

- `None` may or may not be accepted depending on the tool and SDK behavior.
- Invalid argument shapes are left to the MCP SDK/server to reject.
- If the SDK mutates the mapping, that happens outside this function’s control.

## Return Value

Returns a `str` produced by:

```python
"\n".join(parts)
```

Where `parts` contains either each content block’s `.text` value or `str(content)`.

Consequences:

- Empty `result.content` returns `""`.
- Blank strings are preserved and can produce leading, trailing, or repeated newlines.
- There is no structured result, metadata, image payload, resource object, or error flag returned to the caller.
- Any exceptions raised before the join are propagated; they are not converted into return values.

Because the normal caller is `McpBridge.call_tool`, the synchronous caller receives this string from `future.result(timeout=60)`. That caller must be prepared to handle MCP/SDK exceptions and timeout errors.

## Algorithm

1. Await `self._session.call_tool(name, arguments)`.
   - This performs the actual MCP request over the session’s transport, normally stdio.
   - Execution yields to the background event loop while waiting.

2. Create an empty list `parts`.

3. Iterate over `result.content`.
   - For each content object, check `hasattr(content, "text")`.
   - If true, append `content.text` directly.
   - If false, append `str(content)`.

4. Join all collected pieces with a newline:

   ```python
   return "\n".join(parts)
   ```

This means text-bearing MCP content blocks become their text, while non-text blocks become their Python string representation.

## Side Effects

- Performs external I/O by sending an MCP tool call over the client session.
- Executes the tool on the MCP server, which may have arbitrary external side effects.
- Calls `str(content)` for non-text content blocks, which may invoke custom `__str__` behavior.
- Allocates and fills a local `parts` list.
- Does not modify `self`, `_tools`, `_instructions`, `_session`, or other bridge fields.
- Does not cancel or clean up anything if it fails.

Concurrency note: multiple synchronous callers could call `McpBridge.call_tool`, each scheduling this coroutine on the same event loop. This method has no lock or serialization; concurrent safety depends on `ClientSession` and the MCP server.

## Error Handling

This method has no `try`/`except` block. Errors propagate to the caller, usually through the `concurrent.futures.Future` returned by `run_coroutine_threadsafe`.

Possible failures include:

- `AttributeError` if `self._session` is `None`.
- MCP/SDK/transport exceptions from `self._session.call_tool(...)`.
- `AttributeError` if `result` lacks `content`.
- `TypeError` if `result.content` is not iterable.
- `TypeError` if `"\n".join(parts)` receives a non-string value, such as `None`, from a content object whose `text` attribute exists but is not a string.
- Exceptions from content iteration or from `str(content)`.

The method does not:

- Inspect `result.isError`.
- Convert tool errors into structured return values.
- Log failures.
- Apply a timeout.
- Cancel the underlying coroutine if the sync caller times out.

The 60-second timeout lives in `McpBridge.call_tool` via `future.result(timeout=60)`, not in `_call_tool_async`.

## Usage Patterns

Typically this method is not called directly. It is used as:

```python
with McpBridge("some-mcp-server-command") as bridge:
    text = bridge.call_tool("tool_name", {"arg": "value"})
```

Caller obligations:

- Call `connect()` before any `call_tool`.
- Keep the bridge alive until the call completes.
- Do not call after `close()`.
- Handle MCP/SDK exceptions and timeouts.
- Treat the return value as flattened text, not as a full MCP `CallToolResult`.
- Avoid directly awaiting `_call_tool_async` on a different event loop because the session is tied to the bridge’s background loop.

## Dependencies

Internal dependencies:

- `McpBridge._session`
- `McpBridge.call_tool`, which schedules this coroutine
- `McpBridge._session_lifecycle`, which establishes the live MCP session
- `McpBridge.close`, which shuts down the session/event loop

External dependencies:

- `mcp.ClientSession.call_tool`
- MCP content block types returned in `result.content`
- Python built-ins: `hasattr`, `str`, `"\n".join`

## Topics to Explore

- [file] `reasons/mcp_client.py` — Full `McpBridge` lifecycle, including connection setup, background loop management, and shutdown.
- [function] `reasons/mcp_client.py:McpBridge.call_tool` — Shows how this coroutine is scheduled and how the 60-second synchronous timeout is applied.
- [function] `reasons/mcp_client.py:McpBridge._session_lifecycle` — Establishes the `ClientSession`, initializes it, and lists tools before calls are allowed.
- [general] `mcp-call-tool-result-content` — MCP content block variants and how text, image, and resource payloads are represented in `CallToolResult`.
- [general] `mcp-session-concurrency` — Whether `ClientSession.call_tool` supports concurrent calls scheduled on the same event loop.

## Beliefs

- `call-tool-async-returns-newline-joined-content` — `_call_tool_async` returns one string produced by joining each `result.content` item with `"\n"`.
- `call-tool-async-requires-live-session` — It assumes `self._session` is a live initialized MCP `ClientSession` bound to the bridge’s background event loop.
- `call-tool-async-propagates-exceptions` — Exceptions from `ClientSession.call_tool`, content iteration, or the final join are not caught or wrapped.
- `call-tool-async-does-not-apply-timeout` — Timeout enforcement exists only in `McpBridge.call_tool` via `future.result(timeout=60)`.
- `call-tool-async-assumes-text-is-string` — If a content object has a `text` attribute, its value is appended directly without checking that it is a string.
