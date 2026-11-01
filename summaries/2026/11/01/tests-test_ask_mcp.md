---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_ask_mcp.py

# `tests/test_ask_mcp.py`

## 1. Purpose

This file is the integration test suite for **MCP (Model Context Protocol) server support in the `ask` command**. The `ask` command lets a user pose a natural-language question to a belief store and get an LLM-mediated answer. MCP servers extend that answer loop with *external tools* (e.g., "run a SQL query against Snowflake") that the LLM can invoke mid-conversation.

This test file owns three responsibilities:

- **Prompt assembly correctness** — verifying that MCP tool definitions and server-specific instructions are rendered into the LLM prompt in the right format.
- **Tool-call extraction** — confirming the parser still works whether the LLM emits a built-in tool call (`search_beliefs`) or an MCP tool call (`execute_query`).
- **End-to-end dispatch** — confirming that `ask()` actually routes an MCP tool call to the bridge, feeds the result back into the loop, handles errors gracefully, and respects the bumped iteration cap.

## 2. Key Components

### `FakeBridge` (lines 19–35)

A minimal stand-in for the real `McpBridge`. Its contract:

| Method | Return | Notes |
|--------|--------|-------|
| `list_tools()` | `list[dict]` | Each dict has `name`, `description`, `input_schema` (with `properties`) |
| `get_instructions()` | `str` | Free-text guidance for the LLM; empty string means "no instructions" |
| `call_tool(name, arguments)` | `str` (JSON) | Simulates a successful tool invocation |

The class deliberately mirrors the *duck-type* interface that `reasons.ask` expects from any bridge, so the tests never touch a real MCP server.

### `TestBuildToolsSection`

Verifies `_build_tools_section(bridges)`:

- **`test_no_mcp_servers`** — with an empty bridge list, the section still contains the default `search_beliefs` tool.
- **`test_includes_mcp_tools`** — each MCP tool's `name`, `description`, and parameter schema appear in the rendered text.
- **`test_multiple_servers`** — tools from two different bridges both appear.
- **`test_no_params_no_trailing_comma`** — a formatting guard: when a tool has zero parameters, the JSON example is `'{"tool": "list_tables"}'` *not* `'{"tool": "list_tables", }'`. This is a common LLM-prompt bug (trailing commas confuse strict JSON parsers).

### `TestBuildMcpInstructions`

Verifies `_build_mcp_instructions(bridges)`:

- Empty instructions on all bridges → returns `""`.
- Multiple bridges with non-empty instructions → all strings are concatenated into the result.

### `TestAskPromptWithMcp`

Verifies `build_ask_prompt(question, context, tools_section=..., mcp_instructions=...)`:

- Default call produces a prompt containing "one tool available" and the `search_beliefs` JSON example.
- Passing a custom `tools_section` replaces the default.
- Non-empty `mcp_instructions` causes a `"Data Source Instructions"` heading to appear in the prompt.
- Empty `mcp_instructions` suppresses that section entirely.

### `TestExtractToolCallMcp`

Verifies `extract_tool_call(text)`:

- A JSON string with an MCP tool name (`execute_query`) is parsed and returned as a dict.
- The built-in `search_beliefs` path still works (regression guard).

### `TestAskWithMcpDispatch`

The heaviest class — exercises the full `ask()` loop with a mocked LLM:

- **`test_mcp_tool_dispatched`** — Sees the LLM return `'{"tool": "run_query", "sql": "SELECT 1"}'` on the first turn, asserts the bridge's `call_tool` is invoked with `("run_query", {"sql": "SELECT 1"})`, and that the second LLM turn (the final answer) is returned to the caller.
- **`test_mcp_tool_error_handled`** — The bridge raises `RuntimeError("connection lost")`. The test asserts the exception is *swallowed* (captured in tool history) and the loop continues to the next LLM call, which produces a final answer.
- **`test_max_iterations_bumped_with_mcp`** — When `mcp_servers` is non-empty, the iteration cap is 5 (vs. the default of 3). The mock LLM returns tool calls 5 times then a final answer; the test asserts exactly 6 `invoke_model` calls were made (5 tool turns + 1 answer).

### `run_cli` helper (lines 113–124)

A small utility that invokes `reasons.cli.main()` with a patched `sys.argv`/`sys.stdout`/`sys.stderr`. Used to set up a real SQLite database (`init`, `add`) before the dispatch tests, so the belief store isn't empty.

### `db_path` fixture

Points to a temporary SQLite file via `tmp_path`, ensuring each test gets an isolated database.

## 3. Patterns

| Pattern | Where |
|---------|-------|
| **Test double / Duck-typing** | `FakeBridge` implements the same interface as the real `McpBridge` without importing it. Tests depend on the *shape* of the interface, not the concrete class. |
| **Scripted LLM responses** | The `idx` list + `mock_invoke` closure simulates a multi-turn conversation deterministically (turn 1 → tool call, turn 2 → answer). Avoids any network/LLM dependency. |
| **Call tracking** | `test_mcp_tool_dispatched` wraps `bridge.call_tool` in a `tracking_call` closure to assert *what* was called and *with what args*, without breaking the return value. |
| **CLI-driven fixture setup** | `run_cli("init", ...)` + `run_cli("add", ...)` exercises the real storage path instead of hand-crafting DB rows, catching schema drift early. |
| **Negative formatting assertions** | `test_no_params_no_trailing_comma` asserts both the positive form *and* the absence of the malformed form — a common trick for prompt-formatting tests. |

## 4. Dependencies

**Imports (what this file depends on):**

- `reasons.ask` — `_build_tools_section`, `_build_mcp_instructions`, `build_ask_prompt`, `extract_tool_call`, `ask` (the functions under test)
- `reasons.cli` — `main` (to bootstrap a real DB)
- `unittest.mock.patch` — to stub `sys.argv`, `sys.stdout`, `sys.stderr`, and `reasons.ask.invoke_model`
- `pytest` — test runner + `tmp_path` fixture
- `json`, `sqlite3`, `io.StringIO` — utility imports (note: `sqlite3` and `json` are imported but only `json` is used by `FakeBridge.call_tool`; `sqlite3` appears to be a vestigial import)

**Imported by (what depends on this file):**

Nothing. It's a leaf test file. It is *discovered* by pytest via the `tests/` package but not imported by any other module.

## 5. Flow

```
pytest collects the module
│
├─ TestBuildToolsSection (pure functions, no DB)
│   └─ calls _build_tools_section([FakeBridge(...)]) → assert substrings
│
├─ TestBuildMcpInstructions (pure functions)
│   └─ calls _build_mcp_instructions([FakeBridge(...)]) → assert substrings
│
├─ TestAskPromptWithMcp (pure functions)
│   └─ calls build_ask_prompt(...) → assert prompt content
│
├─ TestExtractToolCallMcp (pure functions)
│   └─ calls extract_tool_call(json_string) → assert parsed dict
│
└─ TestAskWithMcpDispatch (integration, needs DB)
    ├─ fixture db_path → tmp_path / "test.db"
    ├─ run_cli("init")          → creates SQLite DB + tables
    ├─ run_cli("add", ...)      → inserts one belief row
    ├─ construct FakeBridge
    ├─ patch reasons.ask.invoke_model with scripted responses
    ├─ call ask(question, db_path, mcp_servers=[bridge])
    │   └─ internally:
    │       1. build prompt with tools_section + mcp_instructions
    │       2. invoke_model → LLM returns tool-call JSON
    │       3. extract_tool_call parses it
    │       4. dispatch: if tool is MCP → bridge.call_tool(...)
    │       5. append tool result to conversation
    │       6. loop (up to max_iterations) until LLM returns non-tool text
    │       7. return final answer string
    └─ assert on result, call count, and bridge.call_tool args
```

The critical data transformation is: **LLM text → JSON dict → bridge.call_tool → JSON result string → appended to prompt → LLM text (final answer)**. The tests verify each hop.

## 6. Invariants

1. **Default tool always present**: `_build_tools_section` must always include `search_beliefs` regardless of how many MCP bridges are passed. The MCP tools are *additive*, never replacing the built-in.

2. **No trailing comma in zero-param tools**: The rendered JSON example for a tool with empty `properties` must be `'{"tool": "name"}'`, never `'{"tool": "name", }'`. This is a hard constraint because downstream LLM JSON parsers (or the LLM itself) may choke on it.

3. **MCP instructions are optional and sectioned**: An empty `mcp_instructions` string suppresses the entire "Data Source Instructions" block. A non-empty one introduces it. There is no partial state.

4. **Error containment in the tool loop**: A `RuntimeError` (or any exception) from `bridge.call_tool` must *not* propagate out of `ask()`. It is captured, the loop continues, and the LLM gets a chance to recover. The test `test_mcp_tool_error_handled` is the guard for this.

5. **Iteration cap scales with MCP presence**: Without MCP servers the loop caps at 3 iterations; with MCP servers it caps at 5. `test_max_iterations_bumped_with_mcp` enforces the "5 tool-turns + 1 answer = 6 total calls" ceiling.

6. **`extract_tool_call` is tool-agnostic**: It must parse *any* valid `{"tool": "<name>", ...}` JSON, whether `<name>` is `search_beliefs` or an arbitrary MCP tool name. No allow-list.

## 7. Error Handling

| Scenario | Behavior | Test coverage |
|----------|----------|---------------|
| `bridge.call_tool` raises `RuntimeError` | Exception is caught inside `ask()`'s dispatch loop, the error message is appended to the tool history in the prompt, and the loop continues to the next LLM turn. `ask()` still returns a string. | `test_mcp_tool_error_handled` |
| LLM returns malformed JSON (not a tool call) | `extract_tool_call` presumably returns `None` or raises; `ask()` treats it as a final answer. (Not explicitly tested here — covered in `test_ask.py`.) | — |
| `invoke_model` times out or fails | Not tested in this file; would be covered by `test_llm.py` or `test_ask.py`. | — |
| `run_cli` hits `SystemExit` | The helper wraps `main()` in `try/except SystemExit: pass` so that the CLI's normal exit-code mechanism doesn't fail the test. | `run_cli` helper |

The key design decision visible here: **the tool dispatch layer is the error boundary**. Errors from the *external* world (bridge/network) are contained and surfaced to the LLM as context, not propagated to the caller. The caller of `ask()` always receives a string (the final answer), never an exception from a tool failure.

---

## Topics to Explore

- [file] `reasons/ask.py` — The module under test. Understanding `ask()`'s loop, how it decides to dispatch to MCP vs. built-in tools, and how it assembles the multi-turn prompt is essential to interpreting these tests.
- [file] `reasons/mcp_server.py` and `reasons/mcp_client.py` — The real MCP bridge implementation. Comparing `FakeBridge`'s interface against the actual `McpBridge` class reveals which methods are guaranteed by the protocol and which are implementation details.
- [function] `reasons/ask.py:build_ask_prompt` — The prompt template that renders tools, instructions, and conversation history. Bugs in template ordering (e.g., instructions appearing after the tool list) would break LLM behavior but pass these substring assertions.
- [file] `tests/test_ask.py` — The non-MCP tests for `ask()`. Reading both files together reveals the full iteration-cap logic and the built-in `search_beliefs` dispatch path that this file's tests assume as a baseline.
- [general] `mcp-protocol-tool-schema` — The MCP spec defines the shape of `input_schema` (JSON Schema subset). Understanding what fields the spec guarantees (`name`, `description`, `input_schema.properties`) helps evaluate whether `FakeBridge`'s fixture data is representative of real-world tool definitions.

---

## Beliefs

- `ask-mcp-dispatch-contains-builtin-search` — `_build_tools_section` always includes the `search_beliefs` tool definition even when one or more MCP bridges are supplied; MCP tools are additive, not a replacement.
- `mcp-tool-error-prevents-ask-raise` — A `RuntimeError` raised by `bridge.call_tool()` is caught inside `ask()`'s dispatch loop and fed back into the prompt as tool history; `ask()` still returns a final answer string rather than propagating the exception.
- `mcp-presence-bumps-iteration-cap-to-5` — When `mcp_servers` is a non-empty list, `ask()` allows up to 5 tool-call iterations before forcing a final answer (vs. 3 iterations in the no-MCP path), verified by exactly 6 `invoke_model` calls.
- `zero-param-tool-renders-no-trailing-comma` — A tool whose `input_schema.properties` is an empty dict must render as `'{"tool": "name"}'` in the prompt; a trailing comma (i.e., `'{"tool": "name", }'`) is explicitly forbidden.
- `fakebridge-implements-mcpbridge-duck-type` — `FakeBridge` is a structural substitute for the real MCP bridge; any code in `reasons.ask` that calls `list_tools()`, `get_instructions()`, or `call_tool(name, args)` will work identically with either, and no test in this file imports the real bridge class.
