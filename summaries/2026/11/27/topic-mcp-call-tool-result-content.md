---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: MCP content block variants and how text, image, and resource payloads are represented in `CallToolResult`.

# MCP Content Blocks in `CallToolResult` — What This Codebase Does (and Doesn't)

## TL;DR

The observed code never references `CallToolResult`, `TextContent`, `ImageContent`, `EmbeddedResource`, `TextResourceContents`, or `BlobResourceContents` by name. A grep for all of those symbols returned **0 matches**, and a symbol/usage search for `CallToolResult` also returned **0**. So the *typed* content-block variants of the MCP protocol are not modeled explicitly anywhere in this repo.

What the repo *does* have is one place that consumes the content: `reasons/mcp_client.py`, in `McpBridge._call_tool_async` (roughly **lines 101–110**). It treats every returned block as either "has a `.text` attribute" or "fallback to `str()`". That's a duck-typed subset of the actual MCP content-block union.

---

## What the code actually does

### 1. The client side — `reasons/mcp_client.py`

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

Key behavior:

- `result` is whatever `ClientSession.call_tool()` returns from `mcp>=1.0`. In the SDK that is a `CallToolResult` with a `.content: list[ContentBlock]` field, plus an `.isError` flag (which this code ignores).
- The loop **flattens** `result.content` into a single `str` joined by `"\n"` — the return type of `_call_tool_async` is `str`, so the caller never sees the block structure.
- Only **one** distinguishing feature is used: `hasattr(content, "text")`.
  - `TextContent` has `.text → str` — this is the happy path.
  - Every other block (images, embedded resources, resource links) hits the `else` branch and is rendered via `str(content)`, which produces the Python `repr` of the pydantic model — **not** the raw bytes or a URL. The `McpBridge.call_tool()` wrapper (lines ~93–97) returns that joined string to the caller.
- `ClientSession.initialize()`-derived tool metadata in `_session_lifecycle` (lines ~78–89) uses `t.inputSchema` (not `input_schema`), which is the SDK's alias and confirms the code is written against the `mcp>=1.0` pydantic naming conventions.

### 2. The server side — `reasons/mcp_server.py`

Every tool registered with `@mcp.tool()` returns **`str`**:

- `search(query, ...) -> str` (returns `api.search(...)`, line ~50)
- `show(node_id) -> str` (returns `json.dumps(...)`, lines ~58–62)
- `add(...) -> str` (returns `json.dumps(...)`, line ~105)
- …and the same pattern for `explain`, `list_beliefs`, `retract`, `assert_belief`, `what_if`, `trace`, `compact`, `propose_*`, `list_proposals`, `show_proposal`, `accept_proposal`.

FastMCP's type adapter takes a plain `str` return value and wraps it in a single `TextContent(text=...)` block inside a `CallToolResult`. **No tool in this server returns `Image` or `Resource` primitives**, so in practice, the result this client ever sees is a `CallToolResult` containing one-or-more `TextContent` blocks.

### 3. The proof that the server only emits text

`tests/test_mcp_server.py` asserts on `json.loads(mcp_server.show("premise-a"))` etc. — the tests parse `str` returns, never touch `CallToolResult` structure. `tests/test_ask_mcp.py` mocks `FakeBridge.call_tool` to return a plain string, mirroring the flattening done by `_call_tool_async`. Both confirm the effective contract is *"tool results are strings"*.

---

## How the real MCP content-block variants map onto `CallToolResult`

For the reader's mental model, here is what the MCP spec's `CallToolResult.content: list[ContentBlock]` union contains, and how the code above handles each:

| Variant | Discriminating fields | Payload | Handling in `_call_tool_async` |
|---|---|---|---|
| `TextContent` | `type: "text"`, `text: str` | UTF-8 text | ✅ Taken via `.text` |
| `ImageContent` | `type: "image"`, `data: str` (base64), `mimeType: str` | Image bytes | ⚠️ Falls through to `str(content)` — no `.text` attr, so base64 payload is embedded in a repr string |
| `AudioContent` | `type: "audio"`, `data: str`, `mimeType` | Audio bytes | ⚠️ Same fallback |
| `EmbeddedResource` | `type: "resource"`, `resource: TextResourceContents \| BlobResourceContents` | Full resource payload inlined | ⚠️ Repr'd — for `TextResourceContents` you *lose* the decoded text |
| `ResourceLink` | `type: "resource_link"`, `uri: str`, `name: str` | Pointer, not payload | ⚠️ Repr'd |

The SDK also exposes `CallToolResult.isError: bool`, which this code **does not check**, so a tool error is indistinguishable from a successful text result at the bridge layer (the `search`/`show`-style tools encode errors as `{"error": ...}` in the JSON string instead — see e.g. `mcp_server.show` line ~60).

---

## Gaps worth flagging

1. **Images**: base64 `data` is not extracted. A caller reading the joined string gets a repr like `ImageContent(type='image', data='iVBOR...', mimeType='image/png')`, not usable bytes. If you need to consume an image-producing MCP server, `_call_tool_async` needs an `isinstance(content, ImageContent)` branch that returns `content.data` (or base64-decodes it).
2. **Resource links**: `ResourceLink.uri` is never surfaced cleanly — it will appear in the repr.
3. **`isError`**: not propagated. The bridge cannot distinguish protocol-level errors from successful "error JSON" text.
4. **Structured content**: recent MCP versions add `structuredContent` (a JSON object separate from `content`). Not referenced anywhere here.
5. **Multiple blocks**: joining with `"\n"` conflates a multi-block response into one blob; the caller loses the per-block boundary. The `ask` dispatcher in `reasons/ask.py` (visible via `tests/test_ask_mcp.py`) then treats that blob as a single observation in its tool history.

## What's missing from these observations

- No file directly imports `CallToolResult` from `mcp.types`, so I cannot show you a local type alias or a helper that normalizes content blocks.
- The `reasons/ask.py` loop that consumes `bridge.call_tool(...)` was not included in the read set (only `tests/test_ask_mcp.py` references it), so I can't verify whether *it* re-inspects `isError` or content types. The observations suggest it treats the returned string as opaque text.
- The installed `mcp` version is pinned loosely (`mcp>=1.0` in `pyproject.toml`), so which block variants exist at runtime depends on the resolved version.

---

## Topics to Explore

- [file] `reasons/mcp_client.py` — `_call_tool_async` is the single point where `CallToolResult.content` is consumed; add `isinstance` branches here if you need image/resource handling.
- [function] `reasons/ask.py:ask` — The dispatcher that turns the joined string into tool observations; check whether `isError` or block boundaries should be surfaced to the model.
- [file] `pyproject.toml` — `mcp = ["mcp>=1.0"]` is an open pin; the resolved `mcp.types` determines the exact `ContentBlock` union (structured content was added later).
- [general] `mcp-content-block-handling` — Design of a normalizer returning `list[Text | Image | ResourceRef]` instead of a single joined `str`.
- [general] `tool-error-propagation` — Whether protocol-level `isError=True` should short-circuit the `ask` tool loop instead of being re-prompted.

## Beliefs

- `calltool-result-flattened-to-str-by-bridge` — `McpBridge._call_tool_async` joins all `result.content` blocks into one newline-separated `str`; callers never see the block list.
- `text-content-detected-via-has-text-attribute` — The only content-block discrimination in the codebase is `hasattr(content, "text")` in `reasons/mcp_client.py`.
- `non-text-blocks-rendered-via-str` — `ImageContent`, `EmbeddedResource`, and `ResourceLink` fall through to `str(content)` (pydantic repr), not raw data.
- `mcp-server-tools-return-str` — Every `@mcp.tool()` in `reasons/mcp_server.py` has a `-> str` return type, so FastMCP emits a single `TextContent` block per call.
- `calltool-result-iserror-ignored` — `CallToolResult.isError` is not read anywhere in the observed files; error signalling happens via JSON `{"error": ...}` strings from the server tools.
- `calltool-result-symbol-absent-from-repo` — Grep and symbol search for `CallToolResult` / `TextContent` / `ImageContent` / `EmbeddedResource` / `ResourceLink` returned zero matches in the observed files.
