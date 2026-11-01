---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_ask.py

# `tests/test_ask.py` — Test Suite for the Ask Module

## 1. Purpose

This file is the integration and unit test suite for `reasons/ask.py`, the module that powers the `reasons ask` CLI command. That command answers natural-language questions against a belief network (stored in SQLite) by combining FTS5 text search with an LLM synthesis loop.

The test file owns verifying the **entire ask pipeline**:

- Parsing tool-call JSON from raw LLM output
- Building prompts (tool-enabled, final, simple, natural)
- The multi-round tool-call loop (search → synthesize → search → …)
- Fallback behavior on timeout / crash
- The `--no-synth` compact path (no LLM, just formatted results)
- Source-document augmentation via a separate FTS5 database
- "Natural" mode (strip all belief metadata, remove citation instructions)
- "Dual" mode (run TMS and FTS legs in parallel, then merge)

It's the single most important behavioral contract file for the ask feature.

## 2. Key Components

### Helper: `run_cli(*args, db_path=None)`

```python
def run_cli(*args, db_path=None):
```
Patches `sys.argv`, `sys.stdout`, `sys.stderr`, then calls `reasons.cli.main()`. Returns `(stdout, stderr, exit_code)`. This is the **CLI integration seam** — every test that exercises the full `reasons ask …` command path goes through this. It catches `SystemExit` to capture the exit code.

### `TestExtractToolCall`

Tests `extract_tool_call(text) → dict | None`. The contract under test:
- Scans each line of LLM output for a JSON object.
- Returns the **first** object that contains a `"tool"` key.
- Returns `None` for plain text, malformed JSON, or JSON missing `"tool"`.
- Malformed lines are silently skipped; a valid JSON line later in the text still wins.

### `TestBuildAskPrompt` / `TestBuildSimplePrompt`

Verify prompt construction:
- `build_ask_prompt(question, context, tool_history=None)` — includes the tool schema (`"tool"`, `search_beliefs`) and, when `tool_history` is provided, an "Additional search results" section.
- `build_final_prompt(question, context)` — **no** tool definition (used on the last round).
- `build_simple_prompt(question, context, natural=False)` — no tool definition; toggles between "Cite belief IDs in [brackets]" and "plain natural language" based on `natural`.

### `TestAskNoSynth` / `TestCmdAskNoSynth`

The `no_synth=True` path bypasses the LLM entirely. Tests assert:
- Output lines start with `[IN]` or `[OUT]`.
- `format="markdown"` produces markdown.
- Zero results yields the string `"No results"`.

### `TestAskNoBeliefs`

Covers the **empty-network** and **no-match** edge cases:
- If the LLM returns a "refusal" string, it passes through.
- If `invoke_model` raises `TimeoutExpired` or `RuntimeError` **and** no beliefs were found, the result is `NO_BELIEFS_MSG`.
- Critically, if a tool-call **did** find beliefs before the timeout, the fallback returns those beliefs (not `NO_BELIEFS_MSG`). This is tested in `test_timeout_after_successful_tool_search_returns_beliefs`.

### `TestAskWithMockedLLM`

The core **tool-call loop** tests. The pattern:

```python
def mock_invoke(prompt, model="claude", timeout=300):
    calls[0] += 1
    if calls[0] == 1:
        return '{"tool": "search_beliefs", "query": "retraction"}'
    return "Retraction cascades through dependents [b]."
```

A closure-based call counter sequences LLM responses. Key assertions:
- `test_tool_call_then_answer`: exactly 2 LLM calls (1 tool + 1 final).
- `test_final_tool_call_triggers_extra_synthesis`: if the LLM returns a tool call even on the "final" round, the code makes **one more** call (total 4) and the result must not contain `"search_beliefs"`.
- `test_timeout_returns_search_results`: timeout after a successful search still returns the searched beliefs.
- `test_unknown_tool_returns_response`: an unrecognized tool name is passed through as the raw response.

### `TestAskSourcesPreserved`

Verifies that when a `sources_db` is provided and the LLM makes a tool call, the **second** prompt still contains the `"Source Documents"` section and the actual chunk text. This guards against a regression where tool-call round-trips would drop the source context.

Also tests that `natural=True` strips `**Status:**` and `**Source:**` metadata from tool results before they're fed back into the prompt.

### `TestStripBeliefMetadata`

Thorough tests for `_strip_belief_metadata(context) → str | None`. The function removes:
- `### heading` lines
- `**Status:**`, `**Source:**`, `**Depends on:**`, `**Justification:**`, `**Supported by:**`, `**Supports:**`, `**Depended on by:**`
- `**Related nodes:**` blocks (including the bulleted list)
- `---` separators
- Collapses 3+ consecutive newlines to at most 2

The `test_full_markdown_format` test is the "kitchen sink" — it feeds a realistic multi-belief markdown blob and asserts every metadata token is gone while the belief text remains.

### `TestSearchSourceChunks`

Tests `_search_source_chunks(query, db_path, top_k=N)` against a **real** SQLite FTS5 database (created in the `sources_db` fixture). Covers:
- Matching chunks are returned with filename in the header.
- `section` field appears as `> Section` when present; omitted when `NULL`.
- Single-character query words are filtered out.
- Stop-word-only queries return `""`.
- `top_k` limits result count.
- A nonexistent DB path returns `""` (no crash).

### `TestAskNatural`

Confirms the `natural=True` flag:
- Strips all metadata from the context before it reaches the LLM prompt.
- Replaces the "Cite belief IDs" instruction with "plain natural language".
- Works in both `simple=True` and full (tool-loop) modes.

### `TestAskWithSources`

Tests the `sources_db` parameter in `simple=True` mode:
- Source chunks are appended to the prompt under a `"Source Documents"` heading.
- If beliefs are empty but sources match, the LLM still gets called with sources.
- If **neither** beliefs nor sources match, result is `NO_BELIEFS_MSG`.

### `TestAskDual`

Tests the `dual=True` mode (TMS leg + FTS leg + merge):
- Exactly **3** LLM calls: TMS answer → FTS answer → merged answer.
- `dual=True` **without** `sources_db` raises `ValueError("--dual requires --full-sources")`.
- `natural=True` propagates to the TMS leg (metadata stripped in the first prompt).

### `TestInvokeClaude`

Single test: when `shutil.which("claude")` returns `None`, `_invoke_claude` raises `FileNotFoundError` with a message matching `"claude"`.

## 3. Patterns

| Pattern | Where | Why |
|---|---|---|
| **Stateful mock via closure** | All multi-call tests (`TestAskWithMockedLLM`, `TestAskDual`, `TestAskSourcesPreserved`) | The ask loop calls `invoke_model` multiple times with different expected responses. A closure over a `calls[0]` counter + a response list or conditional is the idiomatic way to sequence mock returns without a full `side_effect=[a, b, c]` list (which would break if the code loops). |
| **CLI integration via `sys` patching** | `run_cli` helper, `TestCmdAskNoSynth` | Tests the real `argparse` → `main()` → `ask()` path without spawning a subprocess. Catches `SystemExit` to get the exit code. |
| **Real SQLite FTS5 fixtures** | `TestSearchSourceChunks.sources_db`, `TestAskSourcesPreserved.sources_db`, `TestAskWithSources.sources_db`, `TestAskDual.sources_db` | FTS5 behavior (tokenization, stop-word filtering, `content=` table sync) is hard to mock. Creating a real virtual table in a temp file is more reliable and tests the actual query path. |
| **`patch("reasons.ask.invoke_model", …)`** | Nearly every LLM test | The mock target is the **imported name** in the `ask` module's namespace, not the original `llm.invoke_model`. This is the correct patching discipline for monkey-patched functions. |
| **`tmp_path` fixture** | `db_path`, all `sources_db` fixtures | Pytest's built-in per-test temp directory. Each test gets an isolated SQLite file. |
| **Table-per-concern test classes** | `TestExtractToolCall`, `TestBuildAskPrompt`, etc. | Groups related assertions; makes `pytest tests/test_ask.py::TestAskWithMockedLLM` a natural way to run a subset. |

## 4. Dependencies

### This file imports (direct)

| Import | Role |
|---|---|
| `reasons.ask` — `extract_tool_call`, `build_ask_prompt`, `build_final_prompt`, `build_simple_prompt`, `ask`, `_invoke_claude`, `_strip_belief_metadata`, `_search_source_chunks`, `NO_BELIEFS_MSG` | The module under test. Note the underscore-prefixed names: tests reach into **private** helpers (`_invoke_claude`, `_strip_belief_metadata`, `_search_source_chunks`), which is a strong signal these are considered semi-public contracts. |
| `reasons.cli` — `main` | CLI entry point for integration tests. |
| `reasons.llm` (indirectly, via `patch("reasons.llm.shutil.which", …)`) | Used once in `TestInvokeClaude`. |
| `sqlite3` | FTS5 fixture creation. |
| `subprocess` | `TimeoutExpired` for simulating LLM timeouts. |
| `unittest.mock.patch` | All LLM mocking. |
| `pytest` | Fixtures, `pytest.raises`. |

### What depends on this file

Nothing imports a test module. However, the **behavioral contracts** asserted here are the de-facto spec for `reasons/ask.py`. Any change to `ask.py` that breaks these assertions breaks the `reasons ask` CLI.

## 5. Flow

The dominant test flow is:

```
1. Create a temp SQLite DB        (db_path fixture / run_cli "init")
2. Seed beliefs                   (run_cli "add" …)
3. [Optional] Create FTS5 sources (sources_db fixture)
4. Patch reasons.ask.invoke_model (with a stateful mock)
5. Call ask(…) with the mode flags (simple, natural, dual, no_synth, sources_db)
6. Assert on:
     - The return string (content, format, NO_BELIEFS_MSG)
     - The number of LLM calls (calls[0])
     - The prompt content passed to the mock (mock_llm.call_args[0][0])
```

For **pure-function tests** (`TestExtractToolCall`, `TestBuildAskPrompt`, `TestStripBeliefMetadata`), the flow is simpler: call the function, assert on the return value. No DB, no mocking.

For **CLI tests** (`TestCmdAskNoSynth`), the flow goes through `run_cli`, which patches `sys.argv`/`sys.stdout`/`sys.stderr`, calls `main()`, and captures the printed output and exit code.

## 6. Invariants

These are the rules the tests **enforce** and that a developer must not violate in `ask.py`:

1. **First tool call wins.** `extract_tool_call` returns the first JSON object with a `"tool"` key; later ones are ignored.
2. **Tool-call JSON must have a `"tool"` key.** JSON objects without it (e.g. `{"name": "test"}`) are not tool calls and return `None`.
3. **Max tool-call rounds is bounded.** The loop allows up to 3 tool calls; on the 4th invocation (the "final" prompt), the LLM must not return a tool call. If it does, the code makes one extra call and the result must **not** contain `"search_beliefs"`.
4. **`NO_BELIEFS_MSG` is returned only when no beliefs were ever found.** If a tool call succeeded before a timeout/crash, the fallback returns the searched beliefs, not `NO_BELIEFS_MSG`.
5. **`dual=True` requires `sources_db`.** Omitting it raises `ValueError`.
6. **`natural=True` strips all structural metadata** (`###`, `**Status:**`, `**Source:**`, `**Depends on:**`, `**Related nodes:**`, `---`) and replaces the citation instruction with "plain natural language".
7. **Source documents survive tool-call round-trips.** The second (or Nth) prompt must still contain `"Source Documents"` and the chunk text.
8. **Single-character and stop-word-only queries return no FTS results.**

## 7. Error Handling

The tests encode a layered fallback strategy:

| Failure | When beliefs found | When no beliefs found |
|---|---|---|
| `subprocess.TimeoutExpired` | Return the search results (belief IDs + text) | Return `NO_BELIEFS_MSG` |
| `RuntimeError` (LLM crash) | Return the search results | Return `NO_BELIEFS_MSG` |
| `FileNotFoundError` (claude not in PATH) | — | `_invoke_claude` raises; caller must handle |
| `ValueError` (dual without sources_db) | — | Propagates to caller |
| Malformed JSON in LLM output | Skipped; next line tried | — |
| Unknown tool name in JSON | Passed through as the raw LLM response | — |
| FTS5 DB path doesn't exist | `_search_source_chunks` returns `""` | — |

The key architectural decision the tests lock in: **errors never crash the CLI**. The `ask()` function catches LLM failures and degrades gracefully to the raw search results or the `NO_BELIEFS_MSG` constant. This is tested explicitly in `TestAskNoBeliefs` and `TestAskWithMockedLLM`.

---

## Topics to Explore

- [file] `reasons/ask.py` — The module under test; understand the actual `ask()` implementation, the tool-call loop, and how `NO_BELIEFS_MSG` is defined to see why the tests assert these specific fallback strings.
- [function] `reasons/ask.py:_strip_belief_metadata` — The regex/line-based parser that removes belief metadata. The tests in `TestStripBeliefMetadata` are effectively its specification; reading the implementation reveals edge cases (multi-line related-nodes blocks, nested markdown).
- [file] `reasons/llm.py` — Provides `invoke_model` (the function every test patches) and `_invoke_claude`. Understanding the real LLM invocation (subprocess call, timeout, model selection) explains why the tests simulate `TimeoutExpired` and `RuntimeError`.
- [file] `tests/test_ask_mcp.py` — The MCP (Model Context Protocol) variant of ask. Reading it alongside this file reveals how the tool-call protocol changes when the LLM is an MCP client instead of a local `claude` binary.
- [general] `dual-mode-architecture` — The `dual=True` path (TMS + FTS + merge) is the most complex code path tested here. Tracing the 3-call sequence and the `ValueError` guard in `ask.py` reveals how the two retrieval backends are composed.

## Beliefs

- `ask-fallback-returns-search-results-on-timeout` — If `invoke_model` raises `TimeoutExpired` or `RuntimeError` after at least one successful `search_beliefs` tool call, `ask()` returns the searched belief text rather than `NO_BELIEFS_MSG`.
- `natural-mode-strips-all-structural-metadata` — When `natural=True`, `_strip_belief_metadata` removes `###` headings, all `**Key:**` fields, `**Related nodes:**` blocks, `---` separators, and the prompt's citation instruction is replaced with "plain natural language".
- `dual-mode-requires-sources-db` — Calling `ask(…, dual=True)` without a `sources_db` raises `ValueError("--dual requires --full-sources")`; the mode is unusable without an FTS5 sources database.
- `extract-tool-call-returns-first-match` — `extract_tool_call` scans lines in order and returns the first JSON object containing a `"tool"` key; subsequent valid tool-call objects in the same LLM response are ignored.
- `test-suites-patch-reasons-ask-invoke-model` — All LLM-dependent tests patch the name `reasons.ask.invoke_model` (the imported binding), not `reasons.llm.invoke_model`, meaning `ask.py` must call `invoke_model` as a module-level name for the mocks to intercept it.
