---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: How `FastMCP.run()` speaks JSON-RPC over stdin/stdout and why stdout must remain clean

## How `FastMCP.run()` serves JSON-RPC over stdin/stdout

The repo does not contain the implementation of `FastMCP` itself — it imports it from the external `mcp` package at `reasons/mcp_server.py:16`:

```python
from mcp.server.fastmcp import FastMCP
```

It then creates the server at `reasons/mcp_server.py:25`:

```python
mcp = FastMCP("reasons")
```

and starts it at `reasons/mcp_server.py:595`:

```python
mcp.run()
```

Because `mcp.run()` is called with no transport argument, the actual transport is selected by FastMCP’s default. In the MCP Python SDK, that default is the stdio transport: the server reads JSON-RPC messages from `stdin` and writes JSON-RPC responses to `stdout`.

The flow is roughly:

1. The client launches the server as a subprocess.
2. The client writes JSON-RPC requests to the server process’s `stdin`.
3. FastMCP parses those requests and dispatches them to registered handlers.
4. FastMCP writes JSON-RPC responses back to the server process’s `stdout`.
5. The client reads and parses those responses from the subprocess’s `stdout`.

This is confirmed by the client-side code in `reasons/mcp_client.py`. `McpBridge._session_lifecycle` constructs:

```python
server_params = StdioServerParameters(
    command=parts[0], args=parts[1:],
)
```

and then uses:

```python
async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as session:
```

That `stdio_client(...)` pairing is the client half of the stdio transport. `ClientSession` then performs the MCP handshake and tool calls:

- `session.initialize()`
- `session.list_tools()`
- `session.call_tool(name, arguments)` inside `McpBridge._call_tool_async`

So the client side proves the intended contract: the server process is spoken to over its stdio pipes, not over HTTP or a socket.

### What FastMCP does with the registered tools

The `@mcp.tool()` decorators in `reasons/mcp_server.py` register Python functions such as `search`, `show`, `explain`, `add`, `retract`, etc. FastMCP builds tool schemas from their signatures and docstrings. When a client sends a JSON-RPC `tools/call` request, FastMCP:

1. Validates the tool name and arguments.
2. Invokes the matching Python function.
3. Converts the return value into MCP content blocks.
4. Serializes the result as a JSON-RPC response on `stdout`.

For example, `reasons/mcp_server.py:search` returns a string from `api.search(...)`. It does not print that string. FastMCP is responsible for putting the returned string into the protocol response. The same pattern holds for `show`, `explain`, `add`, `retract`, and the other `@mcp.tool()` functions.

On the client side, `McpBridge._call_tool_async` shows how the response is consumed:

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

That is the client-side view of the JSON-RPC `tools/call` response: the server returns content blocks, and the bridge joins their text.

### Why stdout must remain clean

In stdio transport, `stdout` is not a console output stream. It is the server-to-client protocol channel.

That means:

- `stdin` carries JSON-RPC requests from client to server.
- `stdout` carries JSON-RPC responses from server to client.
- `stderr` is available for diagnostics, logs, and errors.

If the server writes anything non-JSON to `stdout` — for example a stray `print(...)`, a startup banner, progress output, or a library that logs to stdout — the client can interpret that text as part of the JSON-RPC stream. At best, the client sees an invalid message. At worst, it loses protocol synchronization and fails the session. The MCP client has no separate control channel for “ignore this line”; `stdout` is the message stream.

This is why the tool functions in `reasons/mcp_server.py` return strings instead of printing them. The grep for `stdout|stdin|stdio|print\(|logging|logger` in `reasons/mcp_server.py` returned zero matches, which is consistent with a module that deliberately avoids writing directly to stdout. Writing to `stderr`, or using a logger configured to use `stderr`, would be the safe way to emit diagnostics.

The tests in `tests/test_mcp_server.py` import `reasons.mcp_server` and call the tool functions directly, for example `mcp_server.search(...)` and `mcp_server.show(...)`. Those tests do not spawn a subprocess or exercise the stdio transport, so they validate the tool logic but not stdout cleanliness or JSON-RPC framing.

### Limitation of the observations

The exact internals of `FastMCP.run()` are not in this repo. The observations do not include the source of `mcp.server.fastmcp`, the exact default transport implementation, the JSON-RPC framing details, or FastMCP’s logging/error behavior. The explanation above is based on the usage in `reasons/mcp_server.py`, the client-side stdio bridge in `reasons/mcp_client.py`, and the standard MCP stdio contract. To verify the exact wire framing and default transport, the external `mcp` package source or MCP specification would need to be inspected.

## Topics to Explore

- [file] `reasons/mcp_client.py` — Shows the client half of the stdio transport, including subprocess launch and session lifecycle.
- [function] `reasons/mcp_client.py:McpBridge._session_lifecycle` — Demonstrates `StdioServerParameters`, `stdio_client`, `initialize`, and `list_tools`.
- [function] `reasons/mcp_client.py:McpBridge._call_tool_async` — Shows how MCP tool content blocks become returned text.
- [file] `reasons/mcp_server.py` — Contains the `FastMCP` instance, tool registrations, and the `mcp.run()` entrypoint.
- [general] `mcp-python-sdk-fastmcp-run` — External SDK source needed to verify `FastMCP.run()` default transport and exact JSON-RPC framing.

## Beliefs

- `mcp-server-is-fastmcp-instance` — `reasons/mcp_server.py:25` creates `FastMCP("reasons")`, and `reasons/mcp_server.py:595` starts it with `mcp.run()`.
- `mcp-server-uses-default-transport` — `mcp.run()` is called with no transport argument, so transport selection is delegated to FastMCP’s default rather than being configured in this repo.
- `mcp-server-tool-functions-return-strings` — The `@mcp.tool()` functions in `reasons/mcp_server.py` return strings and do not print results directly.
- `mcp-client-requires-stdio-command` — `McpBridge._session_lifecycle` constructs `StdioServerParameters(command=parts[0], args=parts[1:])` and connects via `stdio_client` and `ClientSession`.
- `stdout-cleanliness-is-protocol-requirement` — Because the client reads server `stdout` through `ClientSession`, non-JSON output on the server’s stdout would corrupt the JSON-RPC message stream.
