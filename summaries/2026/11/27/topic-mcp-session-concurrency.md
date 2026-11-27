---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Whether `ClientSession.call_tool` supports concurrent calls scheduled on the same event loop.

## Short answer

The observed code **does allow multiple `_call_tool_async` coroutines to be scheduled onto the same background event loop**, but it **does not prove that `mcp.ClientSession.call_tool` itself supports concurrent calls safely**.

`McpBridge.call_tool` in `reasons/mcp_client.py:100-104` does:

```python
future = asyncio.run_coroutine_threadsafe(
    self._call_tool_async(name, arguments), self._loop
)
return future.result(timeout=60)
```

That is a thread-safe way to submit work to the loop. If multiple threads call `McpBridge.call_tool` concurrently, multiple `_call_tool_async` coroutines can be queued on `self._loop`. However, `McpBridge.call_tool` is still synchronous from the caller’s perspective because it immediately blocks on `future.result(timeout=60)`. A single thread using the public API cannot naturally overlap calls; it would need multiple threads, direct access to `_call_tool_async`, or a different async wrapper.

Inside `_call_tool_async` at `reasons/mcp_client.py:106-107`, there is no lock, semaphore, queue, or other serialization around:

```python
result = await self._session.call_tool(name, arguments)
```

So the bridge does not prevent concurrent calls from reaching the shared `ClientSession`. Whether that is safe depends on the external MCP SDK’s `ClientSession.call_tool` implementation.

The repository observations do not include the MCP SDK source. The dependency is optional in `pyproject.toml` as `mcp = ["mcp>=1.0"]`, and the imports are from `mcp` at `reasons/mcp_client.py:14`. The only `ClientSession` creation is at `reasons/mcp_client.py:70`, where one session is kept alive inside `_session_lifecycle`.

The tests also do not exercise real concurrent `ClientSession` behavior. `tests/test_ask_mcp.py:34` defines a synchronous `FakeBridge.call_tool`, and the repository-wide search for `asyncio.gather`, `create_task`, `TaskGroup`, or similar patterns returned no matches.

So the precise conclusion is:

- **Can this code schedule multiple tool calls onto the same event loop?** Yes, via repeated `asyncio.run_coroutine_threadsafe` submissions, especially from multiple threads.
- **Does `ClientSession.call_tool` support those concurrent calls?** Unknown from the observations. You would need to inspect the installed `mcp` package’s `ClientSession` and request/response routing implementation, or write a concurrency test using `asyncio.gather` or multiple threads against a real MCP session.

## Topics to Explore

- [file] `reasons/mcp_client.py` — Shows the single shared `ClientSession` lifecycle and the `run_coroutine_threadsafe` wrapper around tool calls.
- [function] `reasons/mcp_client.py:McpBridge.call_tool` — The synchronous bridge method that schedules coroutines on the background loop and blocks on `future.result`.
- [general] `mcp-sdk-client-session-concurrency` — Inspect the installed `mcp` package’s `ClientSession.call_tool` / request-response routing to confirm whether concurrent requests are multiplexed safely.
- [function] `tests/test_ask_mcp.py:FakeBridge.call_tool` — The tests use a synchronous fake, so they do not validate real `ClientSession` concurrency behavior.
- [file] `pyproject.toml` — The MCP dependency is optional (`mcp>=1.0`), so SDK version-specific concurrency semantics matter.

## Beliefs

- `mcp-client-session-is-shared` — A single `ClientSession` instance is created per `McpBridge` and reused for all tool calls in `reasons/mcp_client.py`.
- `bridge-call-tool-blocks-caller` — `McpBridge.call_tool` blocks on `future.result(timeout=60)` after scheduling the coroutine on the background loop.
- `bridge-schedules-calls-on-background-loop` — `McpBridge.call_tool` uses `asyncio.run_coroutine_threadsafe` to submit each `_call_tool_async` coroutine to `self._loop`.
- `bridge-does-not-serialize-tool-calls` — There is no lock, semaphore, or queue around `_session.call_tool` in `McpBridge`.
- `tests-do-not-exercise-concurrent-client-session` — The observed tests use `FakeBridge.call_tool` and contain no `asyncio.gather`, `create_task`, or `TaskGroup` usage.
