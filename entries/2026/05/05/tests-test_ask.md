# File: tests/test_ask.py

**Date:** 2026-05-05
**Time:** 15:24

I'll work from the test file content you provided — it reveals the production API surface thoroughly through its assertions.

---

# `tests/test_ask.py` — Explanation

## 1. Purpose

This file is the test suite for `reasons_lib.ask`, the **query/search module** of the reasons TMS. The `ask` module lets users pose natural-language questions against a belief network and (optionally) an FTS5-indexed source document database. It supports three execution modes: full agentic (LLM with tool-use loop), simple (single LLM call), and no-synth (raw search results, no LLM). The tests verify every combination of mode, error condition, and output format.

## 2. Key Components

### Test Classes (12 total)

| Class | What it covers |
|---|---|
| `TestExtractToolCall` | Parsing JSON tool calls from LLM text output |
| `TestBuildAskPrompt` | Prompt construction for the agentic (tool-use) path |
| `TestBuildSimplePrompt` | Prompt construction for the simple (no tool-use) path |
| `TestAskNoSynth` | `ask(..., no_synth=True)` — raw FTS5 results, no LLM |
| `TestCmdAskNoSynth` | CLI `reasons ask --no-synth` end-to-end via `run_cli` |
| `TestInvokeClaude` | Guard: `_invoke_claude` raises `FileNotFoundError` when `claude` binary is missing |
| `TestAskNoBeliefs` | Behavior when the network is empty or query matches nothing |
| `TestAskWithMockedLLM` | Full agentic loop: direct answers, tool calls, retries, timeouts |
| `TestAskSourcesPreserved` | Source documents survive across tool-call iterations |
| `TestStripBeliefMetadata` | `_strip_belief_metadata` removes markdown headers, status lines, separators |
| `TestSearchSourceChunks` | `_search_source_chunks` FTS5 queries against the source-chunks DB |
| `TestAskNatural` | `natural=True` mode strips metadata and cite instructions from prompts |
| `TestAskWithSources` | `sources_db` integration — sources appended to prompt context |
| `TestAskDual` | `dual=True` mode — three LLM calls (TMS leg, FTS leg, merge) |

### Helper: `run_cli(*args, db_path=None)`

A test utility that invokes the `reasons` CLI in-process by patching `sys.argv`, `sys.stdout`, and `sys.stderr`. Returns `(stdout, stderr, exit_code)`. Used throughout to set up belief databases without touching the filesystem CLI.

### Fixture: `db_path`

Creates a fresh temp-directory SQLite path per test. Each test then calls `run_cli("init", db_path=...)` to bootstrap the schema, followed by `run_cli("add", ...)` to insert test beliefs.

### Fixture: `sources_db`

Several test classes create their own `sources_db` fixture that builds a minimal SQLite database with a `chunks` table and a `chunks_fts` FTS5 virtual table. This simulates the external source-document store that `ask` can optionally query.

## 3. Patterns

**Mock-at-the-boundary.** The LLM is always mocked via `patch("reasons_lib.ask.invoke_model", ...)`. Tests never call a real model. The mock target is `invoke_model` (from `reasons_lib.llm`), not `_invoke_claude` — only one test touches `_invoke_claude` directly (to verify the `FileNotFoundError` guard).

**Stateful mock sequences.** Tests that exercise the agentic tool-use loop use a closure with a counter (`calls = [0]`) to return different responses on successive LLM invocations. For example, the first call returns a tool-call JSON, the second returns a final answer. This lets tests verify the exact number of LLM round-trips.

**In-process CLI testing.** `run_cli` patches `sys.argv` and captures stdout/stderr via `StringIO`, catching `SystemExit`. This avoids subprocess overhead and keeps tests fast while still exercising the full CLI dispatch through `main()`.

**Fixtures for database setup.** Each test gets an isolated SQLite database via `tmp_path`. The pattern is always: `run_cli("init")` → `run_cli("add", ...)` → exercise `ask(...)`. Source-doc databases are built with raw `sqlite3` calls since they have a different schema.

**Assertion on prompt content.** Several tests inspect the prompt string passed to the mocked LLM (`mock_llm.call_args[0][0]`) to verify that metadata was stripped, tool definitions were included/excluded, or natural-mode instructions were present.

## 4. Dependencies

**Imports from production code:**
- `reasons_lib.ask` — the module under test: `extract_tool_call`, `build_ask_prompt`, `build_final_prompt`, `build_simple_prompt`, `ask`, `_invoke_claude`, `_strip_belief_metadata`, `_search_source_chunks`, `NO_BELIEFS_MSG`
- `reasons_lib.cli.main` — CLI entry point, used by `run_cli`

**Standard library:** `sqlite3` (fixture DB setup), `subprocess` (for `TimeoutExpired` exception), `sys` / `io.StringIO` (for CLI patching), `unittest.mock.patch`

**Nothing imports this file** — it's a leaf test module.

## 5. Flow

The test file exercises three distinct execution paths through `ask()`:

### No-synth path (`no_synth=True`)
`ask` → FTS5 query on the belief DB → format results (compact or markdown) → return string. No LLM involved. Tested by `TestAskNoSynth` and `TestCmdAskNoSynth`.

### Simple path (`simple=True`)
`ask` → FTS5 query → build prompt via `build_simple_prompt` → single `invoke_model` call → return LLM response. On timeout/error, falls back to raw belief listing. Tested by `TestAskSimple`.

### Full agentic path (default)
`ask` → FTS5 query → build prompt via `build_ask_prompt` (includes tool definition) → `invoke_model` → if response contains `{"tool": "search_beliefs", ...}`, execute the search, append results, rebuild prompt, call LLM again → up to 3 tool-call rounds → on the 4th, switch to `build_final_prompt` (no tool definition) for forced synthesis → return. On timeout/error at any stage, fall back to accumulated belief results. Tested by `TestAskWithMockedLLM`.

### Dual path (`dual=True`)
Three sequential LLM calls: (1) TMS beliefs only, (2) source documents only, (3) merge both answers. Tested by `TestAskDual`.

## 6. Invariants

- **Tool-call budget is 3.** After 3 consecutive tool calls, the system switches to `build_final_prompt` (no tool definition), forcing the LLM to synthesize. `test_final_tool_call_triggers_extra_synthesis` asserts exactly 4 total calls.
- **Graceful degradation.** Any `TimeoutExpired` or `RuntimeError` from the LLM returns accumulated search results rather than crashing. When there are no results at all, `NO_BELIEFS_MSG` is returned.
- **Source documents persist across tool-call rounds.** `test_sources_survive_tool_call` asserts that the prompt on round 2 still contains source-doc content after a tool call on round 1.
- **Natural mode strips all metadata.** When `natural=True`, the prompt must not contain `**Status:**`, `### ` headers, or "Cite belief IDs". It must contain "plain natural language".
- **`_strip_belief_metadata` is idempotent on plain text.** Plain belief text passes through unchanged; only markdown structural elements are removed.
- **`_search_source_chunks` filters single-character and stop words** before building FTS5 queries, preventing empty or over-broad matches.
- **`dual=True` requires `sources_db`.** `ask` raises `ValueError` if `dual=True` without a source database.

## 7. Error Handling

| Error | Source | Handling |
|---|---|---|
| `subprocess.TimeoutExpired` | LLM invocation | Returns accumulated beliefs or `NO_BELIEFS_MSG` if none |
| `RuntimeError` | LLM crash | Same graceful fallback as timeout |
| `FileNotFoundError` | `claude` binary not in PATH | Raised by `_invoke_claude`, tested in `TestInvokeClaude` |
| `ValueError` | `dual=True` without `sources_db` | Raised immediately, tested in `TestAskDual` |
| Bad/missing source DB | `_search_source_chunks` | Returns empty string (swallowed) |
| FTS5 query with only stop words | `_search_source_chunks` | Returns empty string |

The overall strategy is **never crash on LLM failure** — always return something useful (belief listings) or a clear "no beliefs" message.

---

## Topics to Explore

- [file] `reasons_lib/ask.py` — The production implementation of the agentic search loop, prompt construction, and FTS5 querying
- [function] `reasons_lib/ask.py:ask` — The main entry point orchestrating mode dispatch (no-synth vs simple vs agentic vs dual)
- [file] `reasons_lib/llm.py` — Where `invoke_model` and `_invoke_claude` live; the LLM abstraction boundary that tests mock against
- [function] `reasons_lib/storage.py:init` — Database schema setup invoked by `run_cli("init")` — understanding the FTS5 schema is key to understanding search behavior
- [general] `tool-use-loop-budget` — The 3-tool-call budget and forced synthesis on the 4th round is a core design decision worth tracing through the production code

## Beliefs

- `ask-tool-call-budget-is-three` — The agentic `ask` loop permits at most 3 tool-call rounds before forcing synthesis via `build_final_prompt` (no tool definition), resulting in exactly 4 LLM invocations maximum
- `ask-graceful-degradation-on-llm-failure` — When `invoke_model` raises `TimeoutExpired` or `RuntimeError`, `ask` returns accumulated FTS5 belief results rather than propagating the exception; if no results exist, it returns `NO_BELIEFS_MSG`
- `ask-dual-requires-sources-db` — Calling `ask(..., dual=True)` without providing a `sources_db` path raises `ValueError`
- `ask-natural-mode-strips-metadata-and-cite` — When `natural=True`, all belief metadata (`**Status:**`, `### ` headers, `**Source:**`, etc.) is stripped from the prompt context and the "Cite belief IDs" instruction is replaced with "plain natural language"
- `search-source-chunks-filters-stop-words` — `_search_source_chunks` filters out single-character tokens and common stop words before constructing FTS5 queries, returning empty string when no usable terms remain

