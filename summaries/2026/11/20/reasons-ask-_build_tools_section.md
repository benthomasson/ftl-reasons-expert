---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: _build_tools_section in reasons/ask.py

# `_build_tools_section` — Explanation

## Purpose

`_build_tools_section` renders the "tools available" block of the LLM prompt used in `ask()`. It produces a small, human-readable (well, LLM-readable) list of JSON-shaped example calls: one fixed built-in tool (`search_beliefs`) plus one example per MCP tool exposed by every connected MCP bridge. The LLM later reads this section and emits a single JSON line matching one of these examples when it wants to invoke a tool (that line is recovered by `extract_tool_call`).

It exists because the tool loop in `ask()` is driven by **prompted tool calling**, not by the model's native function-calling API. So the prompt itself must advertise the available tool names and their parameter shapes.

## Contract

**Preconditions**
- `mcp_servers` is either `None`, empty, or an iterable of "bridge" objects that implement `list_tools() -> list[dict]`.
- Each element returned by `bridge.list_tools()` **must** be a dict containing at least:
  - `"name"` (string) — used to build the JSON example and to key `mcp_tool_map` downstream.
  - `"description"` (string, possibly `""`) — used for the trailing comment line.
  - `"input_schema"` — a JSON-Schema-shaped dict; may be missing/empty.
- If `input_schema` is present, `properties` (if present) is a mapping `param_name -> param_info`, where `param_info` may contain a `"description"`.

**Postconditions**
- Returns a string whose first three lines are always:
  ```
  You have the following tools available:
  
  {"tool": "search_beliefs", "query": "search terms"}
  ```
  followed by zero or more lines — one JSON example per MCP tool, optionally followed by a `  # first-line description` comment line.
- The returned string has no leading/trailing newline; lines are joined with `"\n"`.
- The built-in `search_beliefs` tool is **always** present, regardless of `mcp_servers`.

**Invariants**
- Every emitted tool line begins with `{"tool": "…"}` — i.e. it is JSON-parseable by `extract_tool_call`, as long as the interpolated descriptions contain no braces/newlines.
- Order of MCP tools follows bridge order, then `list_tools()` return order.

## Parameters

| Name | Type | Meaning |
|------|------|---------|
| `mcp_servers` | `iterable[McpBridge] \| None` | Connected MCP bridges from which to enumerate tools. Falsy (`None`, `[]`) means "no MCP tools"; only the built-in `search_beliefs` example is emitted. |

Edge cases:
- `mcp_servers=None` or `[]` → returns just the 3-line base header + search_beliefs line.
- A tool with **no** properties → emits `{"tool": "<name>"}` (no trailing comma).
- A property with **no** description → placeholder becomes the param name itself: `"param": "<param>"`.
- A tool with **no** description (`""`) → no comment line is appended (the `if tool["description"]:` guard).

## Return Value

Returns the tools section as a single string. The caller (`ask()`) passes it as the `tools_section=` keyword to `build_ask_prompt`, which interpolates it into `ASK_PROMPT`'s `{tools_section}` placeholder. The caller never inspects the string further; it must simply be valid prompt text.

## Algorithm

1. Seed `lines` with the header, a blank line, and the fixed `search_beliefs` example.
2. If `mcp_servers` is truthy, iterate over each `bridge`:
   1. Call `bridge.list_tools()` to get the tool list.
   2. For each `tool`:
      - Pull `input_schema` (default `{}`) and its `properties` (default `{}`).
      - For each `(pname, pinfo)` in properties, append `"<pname>": "<description-or-pname>"` to `param_parts`.
      - Join `param_parts` with `", "` into `example`.
      - If `example` is non-empty:
        `{"tool": "<name>", <example>}`
        else:
        `{"tool": "<name>"}`
      - If the tool has a description, append `"  # " + first_line_of_description`.
3. Return `"\n".join(lines)`.

## Side Effects

- No mutation of arguments, no I/O, no global state changes.
- The only external call is `bridge.list_tools()` — in practice it returns the cached tool metadata from the MCP handshake; it is not expected to perform network I/O inside this function, but the code makes no guarantee of that.

## Error Handling

- **No try/except.** Any exception propagates to `ask()`, which does **not** wrap `_build_tools_section` in a `try`. A failing `list_tools()` aborts the ask call.
- `KeyError` if an MCP tool dict lacks `"name"` or `"description"`.
- Malformed `input_schema` (e.g. `properties` not a dict) → runtime error from `.items()`.
- **Silent footguns**: a parameter description containing `"` or `<`/`>` or a `}` will produce an unbalanced example line, which will not parse in `extract_tool_call` — the code does not sanitize. Only the *tool* description is newline-limited (`split("\n")[0]`); parameter descriptions are **not** newline-limited.

## Usage Patterns

Called exactly once per `ask()` call, and only when MCP servers are present:

```python
if mcp_servers:
    tools_section = _build_tools_section(mcp_servers)
    mcp_instructions = _build_mcp_instructions(mcp_servers)
    for bridge in mcp_servers:
        for tool in bridge.list_tools():
            mcp_tool_map[tool["name"]] = bridge
```

The result then flows into the ask loop's `build_ask_prompt(..., tools_section=tools_section, ...)`. A caller must therefore also maintain the invariant that the tool name string used here matches the one looked up in `mcp_tool_map` — otherwise the LLM's call falls through to the "unknown tool" branch and the raw response is returned.

## Dependencies

- No imports inside the function.
- Requires the duck-typed **`McpBridge`** contract: `list_tools()` returning dicts with `name` / `description` / `input_schema`.
- Depends on downstream code: `build_ask_prompt` (formatter) and `extract_tool_call` (parser that consumes the emitted JSON examples).
- Tests: `tests/test_ask_mcp.py` exercises this most directly.

---

## Topics to Explore

- [function] `reasons/ask.py:ask` — The caller that combines `_build_tools_section` output with the tool loop, `mcp_tool_map`, and `extract_tool_call` parsing.
- [function] `reasons/ask.py:extract_tool_call` — The counterpart that parses the JSON lines `_build_tools_section` produces; valuable for spotting format mismatches.
- [file] `tests/test_ask_mcp.py` — Shows the expected shape of `_build_tools_section` output and MCP bridge behavior.
- [general] `mcp-bridge-interface` — The duck-typed contract (`list_tools`, `call_tool`, `get_instructions`) enforced only implicitly.
- [general] `prompt-tool-protocol` — How this text-based function-calling approach trades reliability for model-portability versus native tool APIs.

## Beliefs

- `build-tools-section-always-injects-search-beliefs` — The section unconditionally includes the `search_beliefs` example even when `mcp_servers` is empty.
- `build-tools-section-requires-name-and-description` — Each MCP tool dict must contain `"name"` and `"description"` keys; missing keys raise `KeyError`.
- `build-tools-section-truncates-descriptions` — Only the first line of an MCP tool description is emitted as the trailing comment.
- `build-tools-section-produces-json-example-lines` — Each emitted tool line is intended to be valid JSON so `extract_tool_call` can recover the invocation.
- `build-tools-section-skips-param-escape` — Parameter descriptions are interpolated into `<…>` placeholders without escaping, so descriptions containing quotes or newlines can break the JSON example.
