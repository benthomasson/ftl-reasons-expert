# File: tests/test_ask_mcp.py

**Date:** 2026-05-11
**Time:** 13:08

I'll work from the test file content you provided, which reveals the source API contracts clearly.

---

# `tests/test_ask_mcp.py` — MCP Server Integration Tests for `ask`

## Purpose

This file tests the MCP (Model Context Protocol) integration layer in `reasons_lib.ask`. The `ask` command is a RAG-like Q&A feature: it builds a prompt from belief context, sends it to an LLM, and lets the LLM call tools in a loop. This test file specifically validates that **external MCP servers** (Snowflake, custom data sources) can be wired into that tool-use loop alongside the built-in `search_beliefs` tool.

It owns three responsibilities:
1. Prompt construction correctly incorporates MCP tool descriptions and instructions
2. Tool call extraction handles both built-in and MCP tool JSON
3. The `ask()` runtime correctly dispatches MCP tool calls to the bridge, handles errors, and respects the elevated iteration limit

## Key Components

### `FakeBridge`

A test double replacing the real `McpBridge` (from `reasons_lib.mcp_client`). It implements the three methods the ask pipeline needs:

- `list_tools()` → returns a list of tool descriptors (name, description, input_schema)
- `get_instructions()` → returns server-provided instructions (e.g., "use mart X for sales data")
- `call_tool(name, arguments)` → returns a JSON string result

This is a hand-rolled fake rather than a `Mock` — it gives the tests full control over return values without `side_effect` wiring.

### `TestBuildToolsSection`

Tests `_build_tools_section(bridges: list)`:

- **No MCP servers**: the default tools section still includes `search_beliefs`
- **With MCP tools**: external tool names and their parameter descriptions appear in the generated prompt text
- **Multiple servers**: tools from all bridges are merged
- **No-params edge case**: tools with empty `properties` produce clean JSON examples without trailing commas (`{"tool": "list_tables"}` not `{"tool": "list_tables", }`)

### `TestBuildMcpInstructions`

Tests `_build_mcp_instructions(bridges: list)`:

- Empty instructions produce an empty string (no noise injected into the prompt)
- Instructions from multiple bridges are concatenated

### `TestAskPromptWithMcp`

Tests `build_ask_prompt(question, context, *, tools_section=None, mcp_instructions=None)`:

- Default prompt includes `"one tool available"` and a `search_beliefs` JSON example
- A custom `tools_section` replaces the default entirely
- `mcp_instructions` are injected under a `"Data Source Instructions"` header
- Empty `mcp_instructions` suppress that header

### `TestExtractToolCallMcp`

Tests `extract_tool_call(text)`:

- MCP tool calls are parsed the same way as built-in ones — the function returns a dict with `"tool"` plus whatever extra keys the LLM included
- No special-casing for known vs. unknown tool names

### `TestAskWithMcpDispatch`

Integration-level tests for `ask()`. These use a real SQLite database (via `run_cli("init")` / `run_cli("add", ...)`), mock only the LLM, and verify:

- **`test_mcp_tool_dispatched`**: The LLM's first response is a tool call JSON → `ask()` dispatches it to the correct bridge via `call_tool` → the tool result is fed back → the LLM's second response is the final answer. Verifies both the return value and that `call_tool` was invoked exactly once with the right arguments.

- **`test_mcp_tool_error_handled`**: When `call_tool` raises `RuntimeError`, the error is captured into the tool history (not propagated) and the LLM gets another chance to respond. This is critical — a flaky external data source shouldn't crash the ask loop.

- **`test_max_iterations_bumped_with_mcp`**: Without MCP servers, the tool-use loop has a lower iteration cap. With MCP servers present, it increases to 5 tool calls before forcing a final answer. The test drives 5 consecutive `search_beliefs` calls, then a final text response on iteration 6, confirming the limit.

## Patterns

**Mock-the-LLM, real-everything-else**: The tests mock `reasons_lib.ask.invoke_model` with a simple index-based responder (`responses[idx[0]]`), but use real SQLite storage and real CLI initialization. This catches integration bugs that pure unit mocks would miss.

**Stateful mock via closure**: The `mock_invoke` functions use a mutable list `idx = [0]` to track call count — a common Python pattern for closures that need mutation without `nonlocal`.

**CLI-as-setup**: `run_cli("init")` and `run_cli("add", ...)` are used to set up test databases. This tests through the real initialization path rather than crafting raw SQLite, which keeps tests resilient to schema changes.

**Monkey-patching on the fake**: In `test_mcp_tool_dispatched`, the test replaces `bridge.call_tool` with `tracking_call` after construction — a lightweight way to spy on calls without a full mock framework.

## Dependencies

**Imports from `reasons_lib.ask`**:
- `_build_tools_section` — prompt fragment generator for available tools
- `_build_mcp_instructions` — prompt fragment generator for server instructions
- `build_ask_prompt` — full prompt assembler
- `extract_tool_call` — JSON parser for LLM tool-call responses
- `ask` — the main ask loop (LLM + tool dispatch)

**Imports from `reasons_lib.cli`**:
- `main` — CLI entry point, used only for test database setup

**Nothing imports this file** — it's a test module.

## Flow

The `ask()` function under test follows this loop:

```
1. Build prompt (beliefs context + tools section + MCP instructions)
2. Call LLM (invoke_model)
3. Extract tool call from response
   ├─ If tool call found:
   │   ├─ If "search_beliefs" → query local belief DB
   │   ├─ If MCP tool → dispatch to matching bridge.call_tool()
   │   │   └─ On error → capture error string, don't raise
   │   └─ Append tool result to history, go to step 2
   └─ If plain text → return as final answer
4. If max iterations reached → force final LLM call
```

The tests validate each branch of this dispatch.

## Invariants

- `_build_tools_section([])` always includes `search_beliefs` — it's the baseline tool, present regardless of MCP configuration.
- `build_ask_prompt` never leaves template placeholders (`{{`) in output.
- `extract_tool_call` returns a dict with at minimum a `"tool"` key.
- MCP tool errors are caught and fed back to the LLM as context, never raised to the caller.
- With MCP servers present, the iteration cap is 5 tool calls (6 total LLM invocations including the final answer).
- Without MCP servers, the iteration cap is lower (implied by the test name and assertion).

## Error Handling

The key error-handling contract tested here: **MCP tool failures are non-fatal**. `test_mcp_tool_error_handled` verifies that when `call_tool` raises `RuntimeError("connection lost")`, the `ask()` loop continues and the LLM produces a final answer. The error message is presumably injected into the prompt history so the LLM can acknowledge the failure, but the test only asserts on the final return value, not the intermediate prompt content.

The `run_cli` helper swallows `SystemExit` — it's only used for setup, and CLI exit codes are irrelevant to what's being tested.

---

## Topics to Explore

- [file] `reasons_lib/ask.py` — The production code under test; see the actual tool dispatch loop, iteration limits, and how `mcp_servers` are threaded through
- [file] `reasons_lib/mcp_client.py` — The real `McpBridge` class that `FakeBridge` stands in for; understand the actual MCP protocol interaction
- [function] `reasons_lib/ask.py:ask` — The main ask loop; trace how `max_iterations` is set based on MCP server presence and how tool results are appended to the prompt
- [file] `tests/test_ask.py` — The non-MCP ask tests; compare to understand what baseline behavior exists without external servers
- [general] `mcp-tool-schema-convention` — How tool descriptors (`name`, `description`, `input_schema`) map to the prompt text the LLM sees, and how the LLM's JSON response is parsed back into a dispatch call

## Beliefs

- `ask-mcp-errors-non-fatal` — When an MCP bridge's `call_tool` raises an exception, `ask()` catches it and continues the tool loop rather than propagating the error to the caller
- `ask-mcp-iteration-limit-is-five` — When `mcp_servers` is non-empty, `ask()` allows up to 5 tool-call iterations before forcing a final LLM response (6 total invocations)
- `build-tools-section-always-includes-search-beliefs` — `_build_tools_section` includes the built-in `search_beliefs` tool in its output regardless of whether MCP bridges are provided
- `ask-prompt-no-template-placeholders` — `build_ask_prompt` never leaves `{{` or `}}` template markers in the generated prompt string
- `extract-tool-call-schema-agnostic` — `extract_tool_call` parses any JSON object with a `"tool"` key identically, with no special handling for known vs. MCP tool names

