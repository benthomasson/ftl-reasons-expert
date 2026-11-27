---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: The caller of `McpBridge.call_tool`; decides whether `McpError` is caught or allowed to bubble.

## Explanation

The production decision point is the caller at `reasons/ask.py:539`:

```python
result = bridge.call_tool(tool_name, args)
```

That call site decides whether an MCP tool-call failure is caught or allowed to bubble. `McpBridge.call_tool` itself does not catch `McpError`.

The propagation path is:

1. `reasons/ask.py:539` calls `bridge.call_tool(tool_name, args)`.
2. `McpBridge.call_tool` in `reasons/mcp_client.py:100-103` schedules `_call_tool_async(...)` on the background event loop and then calls:
   ```python
   return future.result(timeout=60)
   ```
   There is no `try/except` here.
3. `_call_tool_async` in `reasons/mcp_client.py:106-114` awaits:
   ```python
   result = await self._session.call_tool(name, arguments)
   ```
   If the MCP SDK raises `McpError`, it propagates out of this coroutine.
4. `future.result()` re-raises that exception in the caller’s thread, so it surfaces at `reasons/ask.py:539`.
5. The observed search for `McpError` found zero matches, and `reasons/mcp_client.py` does not import `McpError`. So there is no explicit `except McpError` handler visible in the observed code.

What is missing: the surrounding block in `reasons/ask.py` around line 539. The observations include only the matching line, not whether it is inside a `try/except Exception`, a broad fallback, or no handler at all. So the bridge does not decide; the caller does, but the exact caller behavior cannot be confirmed from the provided excerpt.

The test at `tests/test_ask_mcp.py:203` monkeypatches `bridge.call_tool` to throw, which likely exercises this exact path, but the test body is not included in the observations.

## Topics to Explore

- [file] `reasons/ask.py` — Inspect the block around line 539 to see whether `bridge.call_tool` is wrapped in `try/except` and what exception types are caught.
- [file] `tests/test_ask_mcp.py` — Look at the test around lines 168-210; it monkeypatches `call_tool` and throws to verify the caller’s failure behavior.
- [function] `reasons/mcp_client.py:McpBridge.call_tool` — Confirms the bridge only schedules the coroutine and re-raises via `future.result(timeout=60)`.
- [function] `reasons/mcp_client.py:McpBridge._call_tool_async` — The direct `ClientSession.call_tool` await where an SDK `McpError` would originate.
- [general] `mcp-error-semantics` — Determine whether the MCP SDK raises `McpError`, returns an error result object, or both for failed tool calls.

## Beliefs

- `ask-py-is-the-production-caller` — The only production invocation of `McpBridge.call_tool` in the observed matches is `reasons/ask.py:539`.
- `mcp-bridge-call-tool-does-not-catch-mcperror` — `McpBridge.call_tool` has no `try/except` around `future.result`, so it propagates coroutine exceptions to the caller.
- `call-tool-exceptions-cross-thread-boundary-via-future` — `asyncio.run_coroutine_threadsafe` plus `future.result(timeout=60)` re-raises `_call_tool_async` exceptions in the calling thread.
- `mcperror-has-no-explicit-handler` — The observed codebase search found zero `McpError` references, so no explicit `except McpError` handler appears in the matched code.
- `mcp-bridge-call-tool-has-60s-timeout` — `future.result(timeout=60)` can also raise a timeout exception independent of any `McpError` raised by the session.
