---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Understand `CallToolResult.content`, error flags, and how `ClientSession.call_tool` raises or returns tool errors.

# Understanding `CallToolResult.content`, error flags, and `ClientSession.call_tool` behavior

## What the MCP SDK returns

The MCP Python SDK's `ClientSession.call_tool(name, arguments)` returns a `CallToolResult` object. It has (roughly) two fields that matter here:

- `result.content: list[ContentBlock]` — a list of content blocks. Common ones are `TextContent` (which has a `.text` attribute), plus `ImageContent`, `EmbeddedResource`, and others that do *not* have `.text`.
- `result.isError: bool` — a flag that says "the tool ran but reported a failure." Note the semantics: **the RPC succeeded**, the tool chose to report an error.

There are also transport/protocol-level failures that raise rather than return:

- Unknown tool name → typically raises `McpError` (method-not-found style).
- Malformed arguments rejected by the schema → typically raises `McpError`.
- Connection dropped, session closed, protocol decode error → raises.

So the two distinct failure surfaces are:
1. **Returned error** — `isError=True` with error text in `content`. **Not** an exception.
2. **Raised error** — `McpError` / transport exception from the await itself.

## What this code does with the result

`reasons/mcp_client.py:106-113` is the only place the result is consumed:

```python
async def _call_tool_async(self, name, arguments):
    result = await self._session.call_tool(name, arguments)   # line 107
    parts = []
    for content in result.content:                            # line 109
        if hasattr(content, "text"):
            parts.append(content.text)
        else:
            parts.append(str(content))
    return "\n".join(parts)                                  # line 113
```

Observations to draw from:

- **`result.content` is treated as a heterogeneous list** (line 109). The `hasattr(content, "text")` guard on line 110 handles non-text blocks (images, embedded resources) by falling back to `str(content)`. So image content is *not* dropped, but is turned into a string repr — which is essentially useless to the model.
- **`isError` is never checked.** The `grep_is_error` search across the repo returned zero matches inside `mcp_client.py`, and `grep_call_tool_result` returned **zero** matches for the literal `CallToolResult`. The bridge duck-types off `result.content` and never looks at `result.isError`.
- **Consequence**: when a tool returns `isError=True` (e.g., `reasons/mcp_server.py:83` `show()` returns `{"error": "Node 'x' not found"}` inside a normal string — this is a *successful* call, so `isError` is False), or when an MCP server returns `isError=True` with a human-readable message, the bridge silently flattens the error text into the same `"\n".join(parts)` string as a success. The caller in `ask.py` cannot distinguish success from tool-reported failure — it only sees text.

## How errors actually propagate

There are three layers where something can go wrong, and only two of them raise:

**Layer 1 — inside `_call_tool_async` (mcp_client.py:106-113):** any exception from `await self._session.call_tool(...)` (e.g., `McpError` for unknown tool, `ConnectionError`, etc.) propagates up the coroutine.

**Layer 2 — `call_tool` sync wrapper (mcp_client.py:100-104):**

```python
def call_tool(self, name, arguments):
    future = asyncio.run_coroutine_threadsafe(
        self._call_tool_async(name, arguments), self._loop
    )
    return future.result(timeout=60)
```

- `future.result(timeout=60)` **re-raises** whatever `_call_tool_async` raised — so `McpError`, transport errors, etc. surface as Python exceptions here.
- It independently raises `concurrent.futures.TimeoutError` if the tool call exceeds **60 seconds** — this is separate from any MCP error.
- If `_call_tool_async` completes normally (including the `isError=True` case), the string is returned and **no exception is raised**.

**Layer 3 — `reasons/ask.py:539`:**

```python
result = bridge.call_tool(tool_name, args)
```

This is the caller. The name of the test `test_mcp_tool_error_handled` (`tests/test_ask_mcp.py`, near line 200) is literally *"MCP tool errors are captured in tool history, not raised."* — implying `ask()` wraps this call in a `try/except` and records the error in the tool-call history passed back to the model, rather than bubbling it up to the user. **We can't confirm the exact `except` clause** because the observations cut off at line 200 of a 245-line file.

## The practical summary matrix

| Situation | What the SDK does | What `McpBridge.call_tool` does | What `ask.py` sees |
|---|---|---|---|
| Tool succeeds | Returns `CallToolResult(isError=False, content=[...])` | Iterates `content`, joins text | Normal string |
| Tool runs and reports an error | Returns `CallToolResult(isError=True, content=[error text])` | **Ignores `isError`**, joins text | Normal string — error not distinguishable |
| Tool name unknown | Raises `McpError` | Re-raises via `future.result` | Exception — ask.py catch block |
| Bad args against schema | Raises `McpError` | Re-raises | Exception — ask.py catch block |
| Session/transport dies | Raises | Re-raises | Exception — ask.py catch block |
| Tool takes > 60s | (still awaiting) | Raises `TimeoutError` at `future.result` | Exception — ask.py catch block |

## What is missing from the observations

- `reasons/ask.py` lines ~520-560: **we have not seen the `try/except` that the "MCP tool errors are captured" test exercises.** Without it we can only infer the error-handling contract from the test name, not from the code.
- `tests/test_ask_mcp.py` lines 200-245: the body of `test_mcp_tool_error_handled` would confirm whether `ask()` catches `Exception` or specifically `McpError`/`TimeoutError`.
- The MCP SDK source (`mcp.types.CallToolResult`, `mcp.client.session.ClientSession.call_tool`) is not in the observations, so the exact severity/type of raised errors (`McpError` codes) can't be cited from source.

## Topics to Explore

- [file] `reasons/ask.py` — Read around lines 520-560 to see exactly how `bridge.call_tool` exceptions are caught and routed into tool history
- [function] `reasons/mcp_client.py:_call_tool_async` — The single point where `isError` could be checked; currently silently drops the flag
- [file] `tests/test_ask_mcp.py` — Read lines 200-245 to confirm the asserted error-handling contract for `ask()`
- [general] `mcp-sdk-calltoolresult` — The `mcp.types.CallToolResult` shape and `McpError` codes from the SDK, and the JSON-RPC spec for `tools/call` errors
- [function] `reasons/mcp_client.py:call_tool` — The sync wrapper where the 60s `future.result(timeout=60)` timeout is enforced

## Beliefs

- `mcp-client-ignores-calltoolresult-iserror` — `_call_tool_async` never reads `CallToolResult.isError`; tool-reported errors are returned to callers as ordinary text strings
- `call-tool-propagates-exceptions-only` — Only Python exceptions from the awaited `session.call_tool` and from `future.result(timeout=60)` propagate out of `McpBridge.call_tool`; returned `isError=True` results do not raise
- `call-tool-raises-on-60s-timeout` — `call_tool` uses `future.result(timeout=60)` (`reasons/mcp_client.py:104`), which raises `concurrent.futures.TimeoutError` on overrun
- `content-iteration-extracts-text-attribute` — `_call_tool_async` iterates `result.content`, appends `content.text` for blocks that have it, and falls back to `str(content)` otherwise
- `mcp-client-does-not-reference-calltoolresult-type` — The codebase contains no explicit references to the `CallToolResult` type; it duck-types on `.content` and `.text` only
- `ask-catches-tool-exceptions-into-history` — `tests/test_ask_mcp.py` asserts "MCP tool errors are captured in tool history, not raised," implying `ask()` wraps `bridge.call_tool` in a catch block (exact code not in observations)
