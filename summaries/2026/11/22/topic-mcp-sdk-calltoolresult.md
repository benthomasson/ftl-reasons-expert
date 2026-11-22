---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: The `mcp.types.CallToolResult` shape and `McpError` codes from the SDK, and the JSON-RPC spec for `tools/call` errors

# `CallToolResult`, `McpError`, and the `tools/call` Error Contract

## What the observations actually pin down

None of the three greps in this repo hit the SDK's error/result types directly:

- `CallToolResult` — 0 matches
- `McpError|ErrorData` — 0 matches
- `tools/call|isError|structuredContent` — 0 matches

So the SDK types are only used *transitively*, via `FastMCP` (server side) and `ClientSession.call_tool` (client side). The two files that touch them:

- `reasons/mcp_server.py` — a `FastMCP("reasons")` app whose tools return `str` (mostly `json.dumps({...})`), including error cases like `json.dumps({"error": "Node 'x' not found"})`.
- `reasons/mcp_client.py` — a sync bridge whose `_call_tool_async` (near the end of the file, ~lines 117–127) does the only real `CallToolResult` handling in the repo:

```python
async def _call_tool_async(self, name, arguments):
    result = await self._session.call_tool(name, arguments)
    parts = []
    for content in result.content:
        if hasattr(content, "text"):
            parts.append(content.text)
        else:
            parts.append(str(content))
    return "\n".join(parts)
```

That is the *entire* surface between this codebase and the SDK's result/error types. What follows is what those types look like in the `mcp>=1.0` SDK (which `pyproject.toml` declares under `[project.optional-dependencies] mcp = ["mcp>=1.0"]`), plus the JSON-RPC layer underneath — this is the SDK/spec knowledge the reader needs, not something derived from the grep results.

---

## The `mcp.types.CallToolResult` shape

`CallToolResult` is a Pydantic model that mirrors the MCP `tools/call` result object:

```python
class CallToolResult(Result):              # Result adds optional `_meta`
    content: list[ContentBlock]            # required, non-empty in practice
    isError: bool = False                  # tool-level failure flag
    structuredContent: dict[str, Any] | None = None   # newer SDK additions
```

- **`content`** is a *list of content blocks*, not a string. The `ContentBlock` union covers:
  - `TextContent(type="text", text=str, annotations=None)` — the common case
  - `ImageContent(type="image", data=str, mimeType=str)` — base64 payload
  - `AudioContent(type="audio", data=str, mimeType=str)`
  - `EmbeddedResource(type="resource", resource=TextResourceContents | BlobResourceContents)`
  - `ResourceLink(type="resource_link", uri=str, name=str)` in newer SDKs
- **`isError`** is a *tool-level* failure flag. When `True`, the protocol call succeeded — the tool ran and reported a business-logic failure. Clients are supposed to surface the content to the model but treat the result as a failure signal.
- **`structuredContent`** (added for structured tool output) carries a JSON object alongside `content`. If set, the server is expected to also put a JSON mirror in `content[0].text` for backwards compatibility.
- Many SDK versions expose a `result.text` property that joins all `TextContent` blocks; the client here reimplements that manually rather than using the property.

The FastMCP `@mcp.tool()` decorator (imported at the top of `reasons/mcp_server.py`) is what *produces* a `CallToolResult`: a return value of `"some markdown"` becomes `CallToolResult(content=[TextContent(type="text", text="some markdown")], isError=False)`. A raised exception inside the tool body becomes `CallToolResult(content=[TextContent(...)], isError=True)` in most SDK versions.

---

## `McpError`, `ErrorData`, and the integer codes

`McpError` is the exception the Python SDK raises when a JSON-RPC *error response* comes back — i.e. the call itself failed and there is no `result` at all.

```python
class ErrorData(BaseModel):
    code: int
    message: str
    data: Any | None = None

class McpError(Exception):
    error: ErrorData          # .error.code, .error.message, .error.data
```

Codes come from two stacked layers:

| Layer | Code | Constant | Cause |
|---|---|---|---|
| JSON-RPC 2.0 | `-32700` | `PARSE_ERROR` | Malformed JSON on the wire |
| JSON-RPC 2.0 | `-32600` | `INVALID_REQUEST` | Missing/bad `jsonrpc`/`id`/`method` |
| JSON-RPC 2.0 | `-32601` | `METHOD_NOT_FOUND` | Unknown method (e.g. `tools/calll`) |
| JSON-RPC 2.0 | `-32602` | `INVALID_PARAMS` | Bad arguments, schema violation, unknown tool name |
| JSON-RPC 2.0 | `-32603` | `INTERNAL_ERROR` | Server crash mid-request |
| MCP SDK | `-32000` | `CONNECTION_CLOSED` | stdio pipe closed / session terminated |
| MCP SDK | `-32001` | `REQUEST_TIMEOUT` | SDK-imposed request timeout (some versions) |

Two practical rules:

1. **`-32602` is the one a tool server author will care about most** — an unknown `name` in `params.name` or arguments that fail the tool's `inputSchema` produce this. `FastMCP` validates against the JSON schema derived from the Python function signature.
2. **`-32000` is the one a client author will care about most** — it's what `session.call_tool()` raises when the server process dies or the pipe closes mid-flight.

`McpError` is *not* a `CallToolResult` — nothing in `content`/`isError` is populated when this is raised.

---

## JSON-RPC wire format for `tools/call`

Request:
```json
{ "jsonrpc": "2.0", "id": 7,
  "method": "tools/call",
  "params": { "name": "show", "arguments": { "node_id": "premise-a" } } }
```

Success (tool ran, `isError: false`):
```json
{ "jsonrpc": "2.0", "id": 7,
  "result": { "content": [{"type": "text", "text": "{...}"}], "isError": false } }
```

Tool-level failure (tool ran, `isError: true`):
```json
{ "jsonrpc": "2.0", "id": 7,
  "result": { "content": [{"type": "text", "text": "boom"}], "isError": true } }
```

Protocol failure (call never reached a `result`):
```json
{ "jsonrpc": "2.0", "id": 7,
  "error": { "code": -32602, "message": "Invalid params",
             "data": [ {"path": ["arguments","node_id"], "message": "Field required"} ] } }
```

The important architectural point: **`tools/call` has two independent failure channels.** `isError` lives *inside* `result`, `code`/`message` live in the sibling `error` field. They are mutually exclusive in a well-formed response, and consumers must handle both.

---

## How *this* codebase uses (and does not use) that contract

### Server side — `reasons/mcp_server.py`

Every error path is `return json.dumps({"error": "..."})`, e.g.:

- `show()` — `except KeyError: return json.dumps({"error": f"Node '{node_id}' not found"})`
- `retract()`, `assert_belief()`, `trace()`, `what_if()`, `add_justification()`, `nogood()`, all proposal tools — same pattern.

Consequence: **the server never sets `isError=True`.** FastMCP sees a string return and emits `isError: False` with the error string as text content. Callers see a successful tool result whose body happens to be `{"error": ...}`. This is a deliberate-but-loose convention that shows up in the tests:

- `tests/test_mcp_server.py::TestShowTool::test_show_missing_returns_error` asserts `"error" in json.loads(mcp_server.show("nonexistent"))`.
- Every other `*_missing_returns_error` test parses the JSON payload rather than checking a flag.

The server *also* never raises or imports `McpError`/`ErrorData`, so no `-32xxx` code originates from application code. The only `-32602` responses in this repo would come from FastMCP itself rejecting an unknown tool name or a schema violation before the decorated function is even entered.

### Client side — `reasons/mcp_client.py`

`_call_tool_async` (near the end, ~lines 117–127) is the only place in the codebase that observes a `CallToolResult`. It:

- iterates `result.content`
- uses `hasattr(content, "text")` to detect text blocks
- falls back to `str(content)` otherwise (a Pydantic repr — fine for `TextContent`, ugly for `ImageContent`).
- **ignores `result.isError`** — a `{"type":"text","text":"boom"}, isError=True` result would be returned by `call_tool()` indistinguishable from success.
- **ignores `result.structuredContent`** — any structured payload is dropped; only the text mirror survives.
- **does not catch `McpError`** — `call_tool()` at `reasons/mcp_client.py` does `future.result(timeout=60)`, so a `-32000` (server died) or `-32602` (bad args) will propagate as an uncaught exception to `ask.py`.

That's the concrete gap the reader should internalize: the codebase implicitly relies on the "embed errors in JSON content" convention, which works only because `isError` is never inspected — the moment someone flips a tool to `raise`, the client will silently misclassify failures as successes.

---

## What’s missing from the observations

- No installed-copy of `mcp.types` is shown, so the exact `ContentBlock` union and the presence of `structuredContent` depends on the SDK version pinned in the environment. `pyproject.toml` uses `mcp>=1.0` (unpinned); behavior differs notably between `1.0.x` and `1.2+` (`structuredContent` is newer).
- No `mcp.shared.exceptions` or `mcp.types.ErrorCode` source shown, so the `-32000`/`-32001` mapping above is SDK documentation, not verified in this repo.
- No caller of `McpBridge.call_tool` is shown, so we can't see whether upstream code (e.g. `ask.py`) wraps the raw `McpError` — that's a topic for exploration below.

## Topics to Explore

- [file] `reasons/mcp_client.py` — The only file that touches `CallToolResult`; read `_call_tool_async` and `call_tool` to see exactly which fields are read vs. dropped.
- [general] `mcp-sdk-calltoolresult` — Open the installed `mcp.types.CallToolResult` and `ContentBlock` union to see the definitive field list for the pinned SDK version.
- [general] `mcp-sdk-error-codes` — Enumerate `ErrorCode` values (JSON-RPC standard + MCP-specific) to confirm which ranges `FastMCP` actually emits.
- [function] `reasons/mcp_client.py:_call_tool_async` — Where non-text content blocks and `isError` results are silently flattened and could be surfaced to the caller.
- [general] `ask.py-mcp-dispatch` — The caller of `McpBridge.call_tool`; decides whether `McpError` is caught or allowed to bubble.

## Beliefs

- `client-uses-call-tool-result-content-channel` — `_call_tool_async` in `reasons/mcp_client.py` joins text from every item in `result.content` and returns the joined string; it never reads `result.isError` or `result.structuredContent`.
- `client-does-not-catch-mcperror` — `McpBridge.call_tool` calls `future.result(timeout=60)` without a `try/except`, so protocol-level `McpError` exceptions propagate uncaught to the caller.
- `server-embeds-errors-in-tool-content` — Every error branch in `reasons/mcp_server.py` (e.g. `show()`, `retract()`, `what_if()`) returns `json.dumps({"error": "..."})`, causing FastMCP to emit `isError=False` results whose body carries the failure.
- `server-does-not-import-sdk-error-types` — `reasons/mcp_server.py` imports only `FastMCP` from `mcp.server.fastmcp` and never references `McpError`, `ErrorData`, or manual `isError` construction.
- `fastmcp-maps-exceptions-to-is-error` — The `@mcp.tool()` decorator wraps a plain Python function; a normal return becomes `content=[TextContent(text=...)]` with `isError=False`, and a raised exception yields `isError=True` (per SDK behavior, not observed directly in this repo).
- `mcp-dependency-version-is-floating` — `pyproject.toml` declares `mcp = ["mcp>=1.0"]` with no upper bound, so the presence of `structuredContent` and the exact `ErrorCode` set depend on whatever `mcp` version the environment installs.
