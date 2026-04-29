# File: tests/test_ask.py

**Date:** 2026-04-29
**Time:** 17:13

No problem — I have enough from the file content provided to give a thorough explanation.

---

# `tests/test_ask.py` — Test Suite for the Ask Module

## 1. Purpose

This file is the test suite for `reasons_lib.ask`, which implements a **retrieval-augmented question-answering** system over the belief database. The ask module combines SQLite FTS5 full-text search with optional LLM synthesis (via the Claude CLI) to answer natural-language questions about stored beliefs. This test file validates three layers: the tool-call parsing logic, the prompt construction, and the full `ask()` orchestration loop — including all its failure modes.

## 2. Key Components

### `run_cli(*args, db_path=None)`
A test helper that invokes the `reasons` CLI in-process by monkey-patching `sys.argv`, `sys.stdout`, and `sys.stderr`. Returns `(stdout, stderr, exit_code)`. This avoids subprocess overhead while testing the full CLI stack. Used extensively to set up test databases with `init` and `add` commands before exercising `ask`.

### `TestExtractToolCall`
Tests `extract_tool_call()` — a parser that scans LLM output line-by-line looking for JSON objects with a `"tool"` key. The contract:
- Returns the **first** valid JSON object containing `"tool"`, or `None`.
- Silently skips malformed JSON and JSON without a `"tool"` key.
- Empty input returns `None`.

This is the mechanism by which the LLM requests additional searches mid-conversation (an agentic tool-use loop without a formal tool-use API).

### `TestBuildAskPrompt`
Tests `build_ask_prompt(question, context, tool_history=None)` — the prompt assembler. Verified contracts:
- The question and initial search context always appear in the prompt.
- The `search_beliefs` tool definition is always included (so the LLM knows it can request searches).
- When `tool_history` is provided, an "Additional search results" section is appended with prior queries and their results.

### `TestAskNoSynth`
Tests `ask()` with `no_synth=True` — the pure-search path that skips LLM synthesis entirely. This mode returns formatted search results directly. Two format modes are tested:
- **compact** (default): lines like `[IN] prop-bfs — Propagation uses BFS`
- **markdown**: richer formatting with headers or bold text
- When no FTS5 matches are found, returns `"No results"`.

### `TestCmdAskNoSynth`
Tests the same `--no-synth` behavior but through the full CLI entry point (`reasons ask "query" --no-synth`), validating that CLI argument parsing correctly plumbs through to the `ask()` function.

### `TestInvokeClaude`
A single test verifying that `_invoke_claude()` raises `FileNotFoundError` when the `claude` binary isn't on PATH. This is the system boundary check.

### `TestAskWithMockedLLM`
The most important class — tests the full agentic loop with a mocked `_invoke_claude`. Covers six scenarios:

| Test | Scenario | Expected behavior |
|------|----------|-------------------|
| `test_direct_answer` | LLM returns plain text | Return text as-is |
| `test_tool_call_then_answer` | LLM requests a search, then answers | Two LLM invocations; final answer returned |
| `test_max_iterations_forces_answer` | LLM always returns tool calls | Loop terminates; raw LLM response returned (contains `search_beliefs`) |
| `test_timeout_returns_search_results` | `_invoke_claude` raises `TimeoutExpired` | Graceful degradation to raw search results |
| `test_error_returns_search_results` | `_invoke_claude` raises `RuntimeError` | Same graceful degradation |
| `test_unknown_tool_returns_response` | LLM emits unrecognized tool name | Raw response returned (no crash) |

## 3. Patterns

**In-process CLI testing**: `run_cli()` patches `sys.argv`/`sys.stdout`/`sys.stderr` to run the CLI without subprocesses. This is fast and gives access to exit codes, but relies on the CLI's `main()` being import-safe and not calling `os._exit()`.

**Progressive mocking depth**: The suite starts with pure-function unit tests (no I/O), moves to integration tests with a real SQLite database, and only mocks the LLM subprocess boundary (`_invoke_claude`). The database is always real — never mocked.

**Graceful degradation testing**: Multiple tests verify that when the LLM is unavailable (timeout, crash, missing binary), the system falls back to returning raw FTS5 search results rather than raising.

**Agentic loop testing**: `test_tool_call_then_answer` and `test_max_iterations_forces_answer` verify a multi-turn agent loop where the LLM can request additional searches before synthesizing an answer, with a bounded iteration count to prevent infinite loops.

## 4. Dependencies

**Imports from production code:**
- `reasons_lib.ask`: `extract_tool_call`, `build_ask_prompt`, `ask`, `_invoke_claude`
- `reasons_lib.cli`: `main` (for CLI integration tests)

**Standard library:**
- `subprocess.TimeoutExpired` — used as a mock side-effect, not to run processes
- `sys`, `io.StringIO`, `unittest.mock.patch` — for in-process CLI invocation
- `pytest` — test framework and fixtures

**Nothing imports this file** — it's a leaf test module.

## 5. Flow

A typical test in `TestAskWithMockedLLM` follows this data flow:

1. **Setup**: `run_cli("init")` creates a fresh SQLite database with FTS5 tables at `tmp_path/test.db`.
2. **Seed**: `run_cli("add", "belief-id", "belief text")` inserts beliefs so FTS5 has something to match.
3. **Mock**: `patch("reasons_lib.ask._invoke_claude")` replaces the subprocess call to Claude CLI.
4. **Execute**: `ask("question", db_path=db_path)` runs the full pipeline: FTS5 search → prompt construction → LLM call (mocked) → optional tool-call parsing → optional follow-up search → return.
5. **Assert**: Verify the returned string contains expected content and that the mock was called the expected number of times.

## 6. Invariants

- **`extract_tool_call` never raises**: It returns `dict | None`. Malformed JSON is silently skipped.
- **`ask()` never raises on LLM failure**: Timeouts, crashes, and missing binaries all degrade to returning raw search results.
- **The agentic loop is bounded**: `test_max_iterations_forces_answer` proves the loop terminates even when the LLM perpetually requests more searches.
- **First tool call wins**: When multiple JSON tool calls appear in LLM output, only the first is honored.
- **`no_synth=True` never contacts the LLM**: These tests don't mock `_invoke_claude` at all, proving it isn't called.

## 7. Error Handling

The test suite validates a **"never crash, always return something useful"** contract:

- `FileNotFoundError` when Claude CLI is missing — the one case that does raise, tested in `TestInvokeClaude`. This happens at the `_invoke_claude` layer, before the `ask()` try/except.
- `subprocess.TimeoutExpired` — caught by `ask()`, falls back to search results.
- `RuntimeError` — caught by `ask()`, falls back to search results.
- Unknown tool names — not an error; the raw response is returned.
- Max iterations exceeded — not an error; the last LLM response is returned as-is.

The pattern is: `_invoke_claude` is allowed to raise, but `ask()` wraps it and guarantees a string return.

---

## Topics to Explore

- [file] `reasons_lib/ask.py` — The production code this suite tests; contains the agentic loop, FTS5 search integration, and `_invoke_claude` subprocess wrapper
- [function] `reasons_lib/storage.py:init` — How the SQLite database and FTS5 virtual tables are created (the `run_cli("init")` calls depend on this)
- [file] `reasons_lib/cli.py` — The CLI entry point that `run_cli()` invokes; shows how `ask` subcommand arguments (`--no-synth`, `--format`) are parsed and forwarded
- [general] `fts5-search-ranking` — How FTS5 match ranking works and whether the `ask` module uses `rank` or `bm25()` to order results
- [file] `tests/test_cli.py` — Other CLI integration tests that likely use the same `run_cli()` pattern, worth comparing for consistency

## Beliefs

- `ask-never-raises-on-llm-failure` — `ask()` catches `TimeoutExpired` and `RuntimeError` from `_invoke_claude` and returns raw FTS5 search results instead of propagating
- `extract-tool-call-returns-first-match` — When LLM output contains multiple JSON objects with a `"tool"` key, `extract_tool_call` returns only the first one
- `ask-agentic-loop-is-bounded` — The tool-call loop in `ask()` has a maximum iteration count; perpetual tool calls from the LLM are terminated and the raw response is returned
- `no-synth-mode-skips-llm` — When `no_synth=True`, `ask()` returns formatted search results without invoking `_invoke_claude`
- `invoke-claude-raises-when-binary-missing` — `_invoke_claude()` raises `FileNotFoundError` if the `claude` CLI is not on `PATH`, rather than returning an error string

