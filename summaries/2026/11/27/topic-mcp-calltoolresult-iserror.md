---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: How MCP tool-level errors are represented and why this wrapper may return them as text.

## How MCP tool-level errors are represented

There are three distinct layers to separate:

1. **Server-side domain errors are often represented as JSON text, not as MCP errors.**
   In `reasons/mcp_server.py`, many `@mcp.tool()` functions catch expected failures and return a JSON string with an `"error"` key. For example:
   - `show()` catches `KeyError` and returns `json.dumps({"error": f"Node '{node_id}' not found"})`
   - `explain()`, `add()`, `retract()`, `assert_belief()`, `what_if()`, `trace()`, etc. follow the same pattern.
   - `what_if()` has an explicit error return at `reasons/mcp_server.py:193`:
     ```python
     return json.dumps({"error": "text is required for what_if supersede"})
     ```
   The tests under `tests/test_mcp_server.py` assert this behavior, e.g. `TestShowTool.test_show_missing_returns_error`, `TestRetractTool.test_retract_missing_returns_error`, etc. So even before MCP semantics matter, the server is deliberately turning many failures into ordinary text content.

2. **MCP tool-level errors are normally represented on `CallToolResult`.**
   In the MCP SDK, a tool can fail at the tool level while the transport call still succeeds. That is usually represented by a `CallToolResult` with `isError=True` and a `content` list, commonly containing `TextContent`. Transport/protocol failures are different: those raise exceptions.

3. **`McpBridge` flattens all content to a string.**
   In `reasons/mcp_client.py`, `McpBridge.call_tool()` schedules `_call_tool_async()` and returns `future.result(timeout=60)` around lines 98–102. Then `_call_tool_async()` does:
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
   This is around `reasons/mcp_client.py:104-112`. It never checks `result.isError`.

## Why this wrapper may return errors as text

`McpBridge._call_tool_async()` intentionally reduces the structured MCP `CallToolResult` into a single text blob by reading each content item’s `.text` or `str(content)`. That means:

- A normal server response becomes text.
- A server-side JSON error like `{"error": "Node 'x' not found"}` becomes text.
- An actual MCP tool-level error result with `isError=True` and `TextContent` also becomes text, because the bridge ignores the `isError` flag entirely.

So the wrapper loses the MCP error bit. It does not convert `isError=True` into an exception, a dict, or any structured error. Downstream code just sees a string and must parse it or inspect it for an `"error"` key.

There is a separate exception path: if `session.call_tool()` itself raises, or the future times out, then `future.result(timeout=60)` propagates the exception. The test `tests/test_ask_mcp.py:194-195` describes the `ask()` layer as capturing MCP tool errors in tool history rather than raising:

```python
def test_mcp_tool_error_handled(self, db_path):
    """MCP tool errors are captured in tool history, not raised."""
```

That test simulates `bridge.call_tool()` raising `RuntimeError("connection lost")` and asserts `ask()` still returns a useful answer. But note: that is an exception from the bridge, not necessarily an MCP `CallToolResult(isError=True)`. For real MCP tool-level errors, `McpBridge` would not raise at all; it would return the error text because it never inspects `result.isError`.

The observations do not include the implementation of `reasons/ask.py` around tool dispatch/history formatting, so we cannot fully verify exactly how `ask()` records the text. The test docstring and assertions indicate the intent: errors should become part of the tool history rather than crash the ask loop.

## Topics to Explore

- [function] `reasons/mcp_client.py:McpBridge._call_tool_async` — Inspect exactly how `CallToolResult.content` is flattened and where `isError` is ignored.
- [file] `reasons/ask.py` — See how `ask()` catches bridge exceptions and inserts tool results/errors into tool history.
- [file] `tests/test_ask_mcp.py` — Review the MCP dispatch and error-handling tests, especially `test_mcp_tool_error_handled`.
- [general] `mcp-calltoolresult-iserror` — Understand MCP SDK semantics for `isError=True` versus transport exceptions.
- [function] `reasons/mcp_server.py:show` — Example of a server tool converting domain failures into JSON `{"error": ...}` text.

## Beliefs

- `mcp-server-domain-errors-return-json-error-strings` — Tools such as `show`, `explain`, `add`, and `retract` catch expected exceptions and return `json.dumps({"error": ...})` text instead of raising.
- `mcp-bridge-flattens-calltoolresult-content-to-text` — `McpBridge._call_tool_async` joins `result.content` text/string parts with newlines and returns a single string.
- `mcp-bridge-ignores-calltoolresult-iserror` — The bridge never reads `result.isError`, so MCP tool-level errors become normal returned text.
- `mcp-bridge-call-tool-propagates-future-exceptions` — `McpBridge.call_tool` returns `future.result(timeout=60)`, so transport failures or timeouts can still raise to the caller.
- `ask-captures-mcp-bridge-exceptions-in-tool-history` — `tests/test_ask_mcp.py:194-195` verifies that a `RuntimeError` from `bridge.call_tool()` does not prevent `ask()` from returning an answer.
