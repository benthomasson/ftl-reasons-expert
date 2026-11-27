---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Whether `ClientSession` supports concurrent tool calls from multiple threads.

## Direct answer

The observed code does **not** establish that the upstream MCP SDK’s `ClientSession` supports concurrent tool calls from multiple threads. `ClientSession` is not defined in this repository; it is imported from the external `mcp` package at `reasons/mcp_client.py:14` and used as an async context manager at `reasons/mcp_client.py:70`.

What the repo *does* show is that `McpBridge` is designed to let synchronous callers submit tool calls from multiple threads onto a **single background asyncio event loop thread**:

- `McpBridge.connect` creates one event loop and starts one daemon thread: `reasons/mcp_client.py:49-53`.
- `_run_loop` sets that event loop in the background thread: `reasons/mcp_client.py:60`.
- `_session_lifecycle` owns one `ClientSession`: `reasons/mcp_client.py:70`.
- `McpBridge.call_tool` submits coroutines with `asyncio.run_coroutine_threadsafe(...)`: `reasons/mcp_client.py:101-103`.
- The actual call happens in `_call_tool_async`: `await self._session.call_tool(name, arguments)` at `reasons/mcp_client.py:106-107`.

So multiple application threads can call `McpBridge.call_tool`, but the `ClientSession` methods themselves execute only on the background event-loop thread. The scheduling call is thread-safe, but the repo does **not** serialize or lock concurrent `session.call_tool` coroutines. There is no `asyncio.Lock` or `threading.Lock` around `_session.call_tool`.

This means:

- **From multiple caller threads:** yes, `McpBridge.call_tool` is designed to accept submissions from multiple threads.
- **Parallel OS-thread execution of `ClientSession.call_tool`:** no, it runs on one event loop thread.
- **Concurrent in-flight `ClientSession.call_tool` coroutines:** possible, because the wrapper does not serialize them.
- **Safety of those concurrent coroutines:** not proven by the observed code and not covered by tests.

The missing evidence is the upstream `mcp` SDK implementation of `ClientSession.call_tool`, especially how it handles concurrent requests, request IDs, and stream writes. The local concurrency test search also found no tests for `ClientSession`, concurrent calls, threads, or `asyncio.gather` in MCP client tests. Existing tests in `tests/test_mcp_server.py` exercise MCP server tools, not client-side concurrency.

If strict safety is required, the bridge should add an `asyncio.Lock`, semaphore, or request queue around `_call_tool_async`, unless the upstream SDK explicitly documents concurrent `call_tool` support.

## Topics to Explore

- [file] `reasons/mcp_client.py` — Full `McpBridge` implementation showing the single-loop/background-thread design.
- [function] `reasons/mcp_client.py:McpBridge.call_tool` — The synchronous entry point that schedules calls from arbitrary threads.
- [function] `reasons/mcp_client.py:McpBridge._call_tool_async` — The exact point where `ClientSession.call_tool` is awaited; add serialization here if needed.
- [repo] `mcp-sdk` — Upstream MCP SDK source/docs to verify `ClientSession.call_tool` concurrency and thread-safety semantics.
- [general] `mcp-client-concurrency-tests` — Missing tests that would validate multiple threads submitting overlapping tool calls.

## Beliefs

- `client-session-is-external-dependency` — `ClientSession` is imported from `mcp` at `reasons/mcp_client.py:14` and is not defined in this repository.
- `mcp-bridge-uses-single-event-loop-thread` — `McpBridge.connect` creates one asyncio event loop and starts one background daemon thread to run it at `reasons/mcp_client.py:49-53`.
- `call-tool-schedules-on-background-loop` — `McpBridge.call_tool` submits `_call_tool_async` to the bridge loop using `asyncio.run_coroutine_threadsafe` at `reasons/mcp_client.py:101-103`.
- `no-tool-call-serialization-lock` — The observed bridge does not acquire a lock around `self._session.call_tool` in `_call_tool_async` at `reasons/mcp_client.py:106-107`.
- `no-client-concurrency-tests-found` — The concurrency test search for `ClientSession`, `concurrent`, `Thread`, `threading`, and `asyncio.gather` found no matching MCP client tests.
