---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_ask_mcp.py

# `tests/test_ask_mcp.py` — Explanation

## Purpose

This is a **unit/integration test module** for MCP (Model Context Protocol) support inside the `reasons.ask` question-answering pipeline. Its job is to pin down the contract between the agent loop in `reasons.ask` and external MCP servers, covering four concerns:

1. **Prompt assembly** — how available MCP tools and per-server instructions are injected into the ask prompt (`build_ask_prompt`, `_build_tools_section`, `_build_mcp_instructions`).
2. **Tool-call parsing** — how the LLM's JSON tool-call responses are extracted (`extract_tool_call`).
3. **Dispatch** — how `ask()` routes a parsed MCP tool call to the bridge, including argument shaping and error swallowing.
4. **The iteration budget** — that attaching MCP servers raises the max tool-loop iterations to 5.

It does *not* exercise a real MCP server or subprocess; a duck-typed `FakeBridge` stands in for the production `reasons.mcp_client.McpBridge`.

## Key Components

### `FakeBridge`
A minimal stand-in for `McpBridge`. The constructor takes `tools: list[dict]` and `instructions: str`. It exposes the four members the production code depends on:
- `list_tools() -> list[dict]` — each item has `name`, `description`, `input_schema` (with `properties`).
- `get_instructions() -> str`
- `call_tool(name, arguments) -> str` — here returns `json.dumps({"tool": name, "args": arguments, "result": "ok"})`, implying the real bridge returns a JSON *string*, not a dict.

Because it's never subclassed, tests fork behavior per case by *reassigning* `bridge.call_tool` to a wrapper (`tracking_call`) or a throwing lambda.

### `_build_tools_section(bridges: list) -> str`
Builds the tools-description block of the prompt. Contract enforced by tests:
- Always contains the builtin `search_beliefs` tool, even with an empty list (`test_no_mcp_servers`).
- For each MCP bridge/tool, includes the tool name, description, and schema fields (e.g. `"SELECT"` from a `sql` property description).
- Tools with empty `properties` are rendered as `{"tool": "list_tables"}` — **no trailing comma** (`test_no_params_no_trailing_comma` guards a specific formatting bug: `'{"tool": "list_tables", }'` must not appear).

### `_build_mcp_instructions(bridges: list) -> str`
Concatenates `bridge.get_instructions()` across servers. Returns `""` when every server has empty instructions. Multiple servers' text is all present in the result.

### `build_ask_prompt(question, context, tools_section=None, mcp_instructions="") -> str`
Assembles the final prompt. Contract:
- With no `tools_section`, falls back to the default block that mentions `"one tool available"` and embeds `{"tool": "search_beliefs"...`.
- A supplied `tools_section` overrides the default (the default's phrase disappears).
- Non-empty `mcp_instructions` inserts a section header `"Data Source Instructions"`; empty instructions omit it.
- `'{{' not in prompt` guard (`test_default_tools_section`) — confirms no raw template placeholders (Jinja/f-string braces) leak through.

### `extract_tool_call(text) -> dict`
Parses a JSON tool-call blob from model output. Tests only verify it round-trips a dict for both an MCP tool (`execute_query`) and the builtin (`search_beliefs`). It returns a dict whose `"tool"` key names the tool and whose other keys are the arguments.

### `ask(question, db_path=None, mcp_servers=None)`
The agent loop under test. Behavior established here:
- `mcp_servers` is a list of bridges (or bridge-like objects).
- Each tool-call response is dispatched via `bridge.call_tool(name, args)` where `args` is the parsed dict **minus the `"tool"` key** (`test_mcp_tool_dispatched` asserts `("run_query", {"sql": "SELECT 1"})`).
- Loop terminates on a non-tool-call response and returns that string.
- With `mcp_servers` attached, max iterations becomes **5**, so the model may be invoked up to **6** times (5 tool rounds + final answer) — `test_max_iterations_bumped_with_mcp`.

### Helpers
- `db_path` fixture (`tmp_path / "test.db"`) — a per-test SQLite path.
- `run_cli(*args, db_path=None)` — constructs `["reasons", "--db", <path>, *args]`, swaps `sys.argv`/`stdout`/`stderr` via `patch.object`, invokes `reasons.cli.main()`, and swallows `SystemExit` so CLI bootstrap failures don't abort the test. Used to `init` the DB and `add` seed beliefs.

### Test classes
Grouped by unit: `TestBuildToolsSection`, `TestBuildMcpInstructions`, `TestAskPromptWithMcp`, `TestExtractToolCallMcp`, `TestAskWithMcpDispatch`.

## Patterns

- **Duck-typed test double** — `FakeBridge` implements only the interface `ask()` relies on, avoiding real `McpBridge`/subprocess setup. This is a light "fake" (returns canned data) rather than a strict mock.
- **Sequenced mock via `side_effect` + closure counter** — every dispatch test uses an `idx = [0]` list and a `mock_invoke` returning `responses[idx[0]]` then incrementing. This drives a deterministic multi-turn conversation without real LLM calls.
- **Spy wrapper** — `test_mcp_tool_dispatched` records `(name, args)` in a `calls` list by wrapping `bridge.call_tool`, letting the test assert *exactly* what was passed downstream.
- **CLI harness / subprocess-free smoke setup** — `run_cli` exercises the real `argparse` CLI in-process via `patch.object(sys, ...)`.
- **Class-grouped pytest cases** — `Test*` classes organize nested concerns; plain `def test_*` methods use the `db_path` fixture by name.
- **Formatting-invariant guard** — `test_no_params_no_trailing_comma` and the `'{{' not in prompt` check are regression guards for literal string-shape bugs, not just behavioral assertions.

## Dependencies

**This file imports:**
- `reasons.ask`: `_build_tools_section`, `_build_mcp_instructions`, `build_ask_prompt`, `extract_tool_call`, `ask` — the module under test.
- `reasons.cli.main` — invoked via `run_cli` for DB seeding only.
- Stdlib: `json`, `sys`, `io.StringIO`, `unittest.mock.patch`; `pytest` for the fixture and `tmp_path`.
- `sqlite3` is imported but **not referenced** anywhere in the file — vestigial, safe to remove.

**What this file relies on indirectly (but does not import):**
- `reasons.mcp_client.McpBridge` — the production class `FakeBridge` mirrors. Its `call_tool` must return a JSON string and its `list_tools` must return dicts with `name`/`description`/`input_schema.properties`.
- `reasons.storage` / SQLite schema — reached through `reasons.cli.main` (`init`, `add`).
- `reasons.ask.invoke_model` — patched out in dispatch tests; its real signature here is inferred as `invoke_model(prompt, model=..., timeout=...)`.

**What depends on it:** nothing — it's a leaf test module.

## Flow

Triggered indirectly by pytest; representative flow for `TestAskWithMcpDispatch.test_mcp_tool_dispatched`:

1. `run_cli("init", db_path=...)` and `run_cli("add", "a", "Alpha belief", ...)` seed a temp SQLite DB through `reasons.cli.main`.
2. Build a `FakeBridge` exposing one tool `run_query`, and wrap its `call_tool` in `tracking_call` to record dispatch.
3. `mock_invoke` is installed via `patch("reasons.ask.invoke_model", side_effect=mock_invoke)` and returns, in order:
   - turn 1: `'{"tool": "run_query", "sql": "SELECT 1"}'`
   - turn 2: `'The query returned 1."`
4. `ask("run a query", db_path=..., mcp_servers=[bridge])`:
   - builds the prompt (tools section includes `search_beliefs` + `run_query`),
   - calls `invoke_model` → returns a tool call → `extract_tool_call` → `bridge.call_tool("run_query", {"sql": "SELECT 1"})` → the returned JSON string is appended to tool history,
   - loops, calling `invoke_model` again → returns a plain answer → loop exits.
5. Assertions: the final result is `"The query returned 1."`; `calls == [("run_query", {"sql": "SELECT 1"})]`; the tool-round count is 1.

`test_max_iterations_bumped_with_mcp` uses the same shape but a mock that always returns `{"tool": "search_beliefs", ...}` for the first 5 invocations, then a final answer on the 6th — asserting `calls[0] == 6`. This is the test that fixes the "MCP ⇒ 5 iterations" invariant.

The `FakeBridge.call_tool` in `test_mcp_tool_error_handled` is replaced with a lambda that raises inside a generator (`(_ for _ in ()).throw(RuntimeError("connection lost"))`), simulating a transport failure during dispatch; the test then verifies `ask()` still returns a string containing the answer text rather than raising.

## Invariants

- `_build_tools_section` **always** includes `search_beliefs`, regardless of MCP servers.
- Tools with no parameters render without a trailing comma: `{"tool": "<name>"}`, never `{"tool": "<name>", }`.
- When a default tools section is used, it says `"one tool available"` and contains an example call for `search_beliefs`; a caller-supplied `tools_section` fully replaces this.
- The `"Data Source Instructions"` header appears in the prompt **iff** `mcp_instructions` is non-empty.
- Arguments passed to `bridge.call_tool(name, args)` are the parsed tool-call dict minus the `"tool"` key.
- A valid tool call must serialize as JSON with a `"tool"` field; both MCP tools and the builtin `search_beliefs` go through the same `extract_tool_call`.
- With `mcp_servers` attached, the tool-loop budget is 5, so `invoke_model` is called at most 6 times (5 rounds + 1 final).
- The rendered prompt must not contain raw `{{` placeholder tokens.

## Error Handling

- **Exceptions from `bridge.call_tool` do not propagate.** `test_mcp_tool_error_handled` throws `RuntimeError("connection lost")` and asserts `ask()` still returns the final model answer (containing `"alpha"`). The implication is that `reasons.ask` catches tool-dispatch exceptions, records the failure in the tool history, and lets the model continue. This is a deliberate boundary: a broken MCP server degrades the answer rather than crashing the request.
- **CLI `SystemExit` is swallowed** in `run_cli` so `main()`'s argparse/exit paths don't kill the test runner. This means a `SystemExit` from the CLI in these tests is *not* asserted on — the harness is only meant for seeding.
- No `pytest.raises` blocks exist here; all error contracts are asserted by "returns a normal-looking result" rather than by exception type. If future code started propagating MCP errors, `test_mcp_tool_error_handled` would fail because `ask()` would raise before the assertion.

## Topics to Explore

- [file] `reasons/ask.py` — The module under test; inspect `build_ask_prompt`, `_build_tools_section`, `_build_mcp_instructions`, `extract_tool_call`, and the `ask()` loop to see the exact iteration budget logic and how tool errors are folded into history.
- [function] `reasons/ask.py:_build_tools_section` — Understand the exact prompt-rendering format that dictates the no-trailing-comma invariant and how builtin vs. MCP tools are listed.
- [file] `reasons/mcp_client.py` — The real `McpBridge` interface that `FakeBridge` mirrors; confirms `call_tool` returns a JSON string and how `list_tools`/`get_instructions` are shaped.
- [file] `reasons/cli.py` — `main()` and its `init` / `add` subcommands used by `run_cli` for DB seeding; also where `--db` and `SystemExit` behavior live.
- [file] `tests/test_ask.py` — The non-MCP ask tests; highlights the default (non-MCP) iteration limit that `test_max_iterations_bumped_with_mcp` is contrasting against.

## Beliefs

- `mcp-servers-bump-max-iterations-to-five` — When `mcp_servers` is non-empty, `ask()` allows up to 5 tool rounds, so `invoke_model` is called at most 6 times.
- `mcp-tool-errors-are-swallowed` — Exceptions raised by `bridge.call_tool` are caught inside `ask()` (recorded in tool history) and never propagate to the caller.
- `tool-dispatch-arguments-omit-tool-key` — The dict passed as `arguments` to `bridge.call_tool(name, arguments)` is the parsed tool-call minus the `"tool"` key.
- `tools-section-always-includes-search-beliefs` — `_build_tools_section` includes the builtin `search_beliefs` tool even when the bridge list is empty.
- `empty-mcp-instructions-produce-no-prompt-section` — `_build_mcp_instructions` returns `""` for empty instructions, and `build_ask_prompt` then omits the `"Data Source Instructions"` header entirely.
