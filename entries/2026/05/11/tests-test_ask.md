# File: tests/test_ask.py

**Date:** 2026-05-11
**Time:** 13:07

I'll work from the test file content you provided — it's detailed enough to reconstruct the contracts of the module under test.

---

# `tests/test_ask.py` — Tests for the Ask Module

## Purpose

This file tests `reasons_lib.ask`, the query interface that lets users ask natural-language questions against a belief network. The ask module combines **FTS5 full-text search** over beliefs (and optionally over source document chunks) with **LLM synthesis** (via Claude) to produce grounded answers. This test suite validates every mode of that pipeline: no-synth (search only), simple (single-shot LLM), full (multi-turn with tool use), dual (parallel TMS + FTS legs merged), and natural (metadata-stripped, citation-free output).

It's the primary quality gate for the `ask` command — the highest-level read path in the system.

## Key Components

### Test Helpers

- **`db_path` fixture** — Creates a fresh temp-directory SQLite path per test. Every test gets an isolated belief store.
- **`run_cli(*args, db_path=None)`** — Programmatic CLI runner. Patches `sys.argv`/`sys.stdout`/`sys.stderr`, calls `main()`, captures output and exit code. Used to set up belief state (`init`, `add`) rather than calling storage APIs directly — this means tests exercise the full CLI → storage path, not just the ask module in isolation.

### Test Classes (by feature area)

| Class | What it covers |
|---|---|
| `TestExtractToolCall` | Parsing JSON tool-call objects from LLM text output — valid calls, missing `tool` key, malformed JSON, first-wins semantics |
| `TestBuildAskPrompt` | Prompt construction for multi-turn mode — question/context inclusion, tool definition presence, tool history injection |
| `TestAskNoSynth` | `no_synth=True` mode: returns raw FTS results in compact or markdown format, no LLM involved |
| `TestCmdAskNoSynth` | Same as above but through the CLI (`reasons ask --no-synth`) |
| `TestInvokeClaude` | Edge case: `_invoke_claude` raises `FileNotFoundError` when the `claude` binary isn't on `$PATH` |
| `TestAskNoBeliefs` | Behavior when the network is empty or no beliefs match — LLM refusal, timeout/error fallback to `NO_BELIEFS_MSG` |
| `TestAskWithMockedLLM` | Full multi-turn ask loop: direct answer, tool-call-then-answer, max tool iterations (3 calls → forced synthesis on 4th), timeout/error fallback to raw search results, unknown tool passthrough |
| `TestAskSourcesPreserved` | Source document chunks survive across tool-call iterations and appear in subsequent prompts |
| `TestBuildSimplePrompt` | Prompt construction for simple mode — no tool definitions, optional natural/cite modes |
| `TestAskSimple` | `simple=True` mode: single LLM call, no tool loop, graceful degradation on timeout/error |
| `TestStripBeliefMetadata` | `_strip_belief_metadata` correctly removes `### headers`, `**Status:**`, `**Source:**`, `**Depends on:**`, `**Supported by:**`, `**Supports:**`, `**Depended on by:**`, `**Related nodes:**`, separators, and collapses blank lines |
| `TestSearchSourceChunks` | `_search_source_chunks` FTS5 queries against external source databases — matching, section headers, stop-word filtering, top_k, missing DB fallback |
| `TestAskNatural` | `natural=True` mode: metadata stripped from context, cite instructions replaced with "plain natural language" |
| `TestAskWithSources` | `sources_db` parameter: source chunks appended to prompt, sources alone sufficient when no beliefs match, no-match returns `NO_BELIEFS_MSG` |
| `TestAskDual` | `dual=True` mode: three LLM calls (TMS leg, FTS leg, merge), requires `sources_db`, propagates `natural` flag |

## Patterns

**CLI-as-setup**: Tests don't call `storage.add_belief()` directly — they run `run_cli("init")` and `run_cli("add", ...)`. This couples tests to CLI behavior but guarantees the full init/add path works, including FTS index population.

**Mock-at-the-LLM-boundary**: All LLM interaction is mocked via `patch("reasons_lib.ask.invoke_model", ...)`. The mock is a function that counts calls and returns canned responses, allowing tests to verify the multi-turn tool-call loop without hitting a real model. The call counter pattern (`calls = [0]`, `calls[0] += 1`) appears throughout.

**Prompt inspection**: Several tests capture the prompt sent to the mock LLM (`mock_llm.call_args[0][0]`) and assert on its contents — checking that metadata was stripped, tool definitions were included/excluded, or source documents were preserved. This tests the *prompt construction* contract, not just the final output.

**In-memory SQLite fixtures for source chunks**: Tests that need source documents create a fresh SQLite DB with a `chunks` table and `chunks_fts` FTS5 virtual table, fully self-contained.

**Graceful degradation**: A recurring invariant tested across multiple classes: when the LLM times out or crashes, `ask()` falls back to returning raw search results rather than propagating the error. Empty networks get `NO_BELIEFS_MSG`.

## Dependencies

**Imports from `reasons_lib.ask`**: `extract_tool_call`, `build_ask_prompt`, `build_final_prompt`, `build_simple_prompt`, `ask`, `_invoke_claude`, `_strip_belief_metadata`, `_search_source_chunks`, `NO_BELIEFS_MSG`

**Imports from `reasons_lib.cli`**: `main` (used by `run_cli` helper)

**Transitive**: `reasons_lib.llm` (mocked via `shutil.which`), `reasons_lib.storage` (via CLI init/add)

**Nothing imports this file** — it's a leaf test module.

## Flow

A typical test follows this sequence:

1. `run_cli("init", db_path=...)` — create empty belief store with FTS index
2. `run_cli("add", "id", "text", db_path=...)` — populate beliefs (1–2 typically)
3. `patch("reasons_lib.ask.invoke_model", ...)` — mock the LLM
4. `ask("query", db_path=..., <mode flags>)` — exercise the ask function
5. Assert on the returned string and/or the prompt passed to the mock

For multi-turn tests, the mock function uses a call counter to return a tool-call JSON on early calls and a prose answer on the final call, verifying the loop iterates the expected number of times.

## Invariants

- **Tool call loop is bounded**: The full ask mode allows at most 3 tool-call iterations before forcing a final synthesis call (the 4th call uses `build_final_prompt` which omits tool definitions). `test_final_tool_call_triggers_extra_synthesis` enforces `calls[0] == 4`.
- **Graceful degradation on LLM failure**: `TimeoutExpired` and `RuntimeError` from the LLM never propagate to the caller. If beliefs were found by FTS, they're returned as-is. If nothing was found, `NO_BELIEFS_MSG` is returned.
- **Source documents survive tool iterations**: After a tool-call round-trip, the next prompt must still contain the source documents. `test_sources_survive_tool_call` asserts this with `"Source Documents" in prompt`.
- **Natural mode strips all metadata**: `**Status:**`, `**Source:**`, `### headers`, `**Depends on:**`, `**Supported by:**`, `**Supports:**`, `**Depended on by:**`, `**Related nodes:**`, and `---` separators are all removed. Citation instructions are replaced.
- **`dual=True` requires `sources_db`**: Raises `ValueError` otherwise.
- **`extract_tool_call` returns the first valid tool-call JSON**: Malformed lines are skipped, JSON without a `"tool"` key is ignored, and the first match wins.

## Error Handling

The tests verify that `ask()` **absorbs** all LLM errors:

- `subprocess.TimeoutExpired` → returns raw beliefs or `NO_BELIEFS_MSG`
- `RuntimeError` → same fallback
- `FileNotFoundError` from `_invoke_claude` when `claude` binary is missing → raised to caller (this is the only error that propagates, tested in `TestInvokeClaude`)

The pattern is: search always works (it's local SQLite FTS5), synthesis is best-effort. If synthesis fails at any point in the tool-call loop, whatever beliefs have been accumulated so far are returned.

---

## Topics to Explore

- [file] `reasons_lib/ask.py` — The implementation of the multi-turn ask loop, prompt builders, and fallback logic tested here
- [function] `reasons_lib/ask.py:ask` — The main entry point: orchestrates FTS search, prompt construction, tool-call loop, and graceful degradation
- [function] `reasons_lib/ask.py:_strip_belief_metadata` — Regex/line-filtering logic that removes markdown metadata for natural mode
- [file] `reasons_lib/llm.py` — The `invoke_model` function that wraps Claude CLI invocation — mocked throughout these tests
- [file] `tests/test_ask_mcp.py` — Likely tests the MCP-server variant of ask, a related but distinct code path

## Beliefs

- `ask-tool-loop-bounded-at-three` — The multi-turn ask loop allows at most 3 tool-call iterations before forcing a tool-free final synthesis call (4 total LLM invocations)
- `ask-absorbs-llm-failures` — `ask()` never propagates `TimeoutExpired` or `RuntimeError` from the LLM; it falls back to raw FTS results or `NO_BELIEFS_MSG`
- `ask-dual-requires-sources-db` — Calling `ask()` with `dual=True` but no `sources_db` raises `ValueError`
- `ask-natural-strips-all-metadata-fields` — Natural mode removes Status, Source, Depends-on, Supported-by, Supports, Depended-on-by, Related-nodes headers, markdown heading prefixes, and horizontal rules from the belief context before sending to the LLM
- `ask-source-chunks-persist-across-tool-iterations` — Source document content included in the initial prompt is re-included in all subsequent prompts after tool-call round-trips

