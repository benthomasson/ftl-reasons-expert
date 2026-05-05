# File: reasons_lib/ask.py

**Date:** 2026-05-05
**Time:** 15:22

## `reasons_lib/ask.py` — Natural Language Q&A Over a Belief Network

### 1. Purpose

This module is the **question-answering interface** for the TMS belief store. It takes a natural language question, retrieves relevant beliefs via FTS5 full-text search, and optionally synthesizes a human-readable answer using an LLM. It owns the entire retrieval-augmented generation (RAG) pipeline for the `reasons ask` CLI command, including:

- Building prompts for the LLM
- Running a multi-round **tool loop** where the LLM can request additional belief searches
- A **dual-path mode** that independently queries both the belief network and a separate source-document FTS5 index, then merges the answers
- Graceful degradation when the LLM times out or fails

### 2. Key Components

**Prompt templates** — Four string templates define the LLM's behavior:

| Template | Tool available? | Use case |
|---|---|---|
| `ASK_PROMPT` | Yes (`search_beliefs`) | Normal rounds in the tool loop |
| `FINAL_ASK_PROMPT` | No | Last round — forces the LLM to answer instead of searching again |
| `SIMPLE_ASK_PROMPT` | No | Single-pass mode (`simple=True`) — no tool loop at all |
| `FTS_RAG_PROMPT` | No | RAG over source document chunks (not beliefs) |
| `MERGE_PROMPT` | No | Merges TMS and FTS answers in dual mode |

**`ask()`** — The main entry point. Orchestrates the full pipeline with these modes:
- `no_synth=True`: returns raw FTS5 search results, no LLM call
- `simple=True`: single LLM call, no tool loop
- `dual=True`: two independent retrieval paths (TMS + FTS RAG) merged by a third LLM call
- Default: up to `MAX_ITERATIONS` (3) rounds of tool-assisted synthesis

**`extract_tool_call(text)`** — Parses the LLM's text response line-by-line looking for JSON with a `"tool"` key. This is a manual tool-use protocol — no structured tool calling from the API, just convention-based JSON detection in the response text.

**`_strip_belief_metadata(beliefs_context)`** — Removes structural lines (status, justification, dependency info) from belief context, leaving only prose. Used when `natural=True` to present beliefs without TMS jargon.

**`_search_source_chunks(question, sources_db)`** — Queries a separate SQLite FTS5 index (`rag_fts.db`) for document chunks. Tokenizes the question, filters stop words, and runs an OR query. Returns formatted markdown with filename/section headers.

### 3. Patterns

**Manual tool loop** — Rather than using the Anthropic API's native tool-use protocol, this module embeds a tool definition directly in the prompt text and parses JSON from the LLM's response. The loop runs up to 3 iterations. On the final iteration, the prompt switches to `FINAL_ASK_PROMPT` which omits the tool definition, forcing a direct answer.

**Graceful degradation** — Every LLM call is wrapped in try/except for `TimeoutExpired` and generic `Exception`. On failure, the function falls back to returning the raw belief context via `_beliefs_or_no_match()`, so the user always gets something.

**Dual retrieval with late fusion** — The `dual=True` mode runs TMS and FTS RAG independently, then merges with a third LLM call. If either path returns empty, the non-empty answer is used directly without a merge call — saving one LLM invocation.

**Citation mode toggle** — The `natural` flag switches between `_CITE_RULE` (cite belief IDs in brackets) and `_CITE_RULE_NATURAL` (plain language, no IDs). It also triggers `_strip_belief_metadata` to remove structural markup from the context.

### 4. Dependencies

**Imports:**
- `api.search()` — FTS5 belief search (the retrieval half of RAG)
- `api._STOP_WORDS` — stopword list for FTS query construction
- `llm.invoke_model()` — dispatches to the configured LLM backend (Claude, Gemini, etc.)
- `sqlite3` — direct DB access for source-chunk search (bypasses the `api` module)

**Imported by:**
- `reasons_lib/cli.py` — the CLI `ask` subcommand
- `reasons_lib/api.py` — the API layer re-exports or calls into ask
- `tests/test_ask.py` — unit tests

### 5. Flow

**Default (tool-loop) mode:**

```
ask(question)
  → api.search(question) → beliefs_context
  → [optional] _search_source_chunks(question) → sources_suffix
  → for iteration in 0..2:
      → build_ask_prompt() or build_final_prompt() (last round)
      → invoke_model(prompt)
      → extract_tool_call(response)
        → if None: return response (LLM answered directly)
        → if search_beliefs: api.search(query) → append to tool_history
                             update beliefs_context
  → fallback: return _beliefs_or_no_match()
```

**Dual mode:**

```
ask(question, dual=True, sources_db=...)
  → ask(question, ...)  [recursive, TMS path]
  → _fts_rag_answer(question, sources_db)  [FTS path]
  → if one empty: return the other
  → _merge_answers(question, answer_tms, answer_fts)
```

### 6. Invariants

- **`MAX_ITERATIONS = 3`** — the tool loop never exceeds 3 rounds. On the final round, the prompt omits the tool definition.
- **`dual` requires `sources_db`** — enforced with `raise ValueError`.
- **No hallucination by design** — every prompt template includes the rule: "ONLY answer based on the beliefs provided. Do NOT use your training data." Plus a mandatory fallback phrase when beliefs are insufficient.
- **Tool call parsing is line-based** — only lines starting with `{` are candidates. Multi-line JSON tool calls would be missed.
- **`_strip_belief_metadata` preserves blank-line structure** — collapses runs of 3+ newlines to 2 but doesn't collapse all whitespace.

### 7. Error Handling

- LLM timeouts (`subprocess.TimeoutExpired`) and all other exceptions are caught per-call. The fallback is always `_beliefs_or_no_match(ctx)`, which returns the raw belief context if available or `NO_BELIEFS_MSG` if empty.
- `_search_source_chunks` catches `sqlite3.OperationalError` and `DatabaseError`, returning an empty string — so a missing or corrupt `sources_db` silently degrades to belief-only mode.
- If the LLM returns an unrecognized tool name, the response is returned as-is (treated as a direct answer).
- Status messages go to `stderr` so they don't pollute the answer on `stdout`.

---

## Topics to Explore

- [file] `reasons_lib/llm.py` — How `invoke_model` dispatches across LLM backends and what the subprocess call looks like
- [function] `reasons_lib/api.py:search` — The FTS5 search implementation that produces the belief context fed into prompts
- [file] `tests/test_ask.py` — Test coverage for the tool loop, dual mode, and edge cases like empty results and timeouts
- [function] `reasons_lib/ask.py:_strip_belief_metadata` — The regex and line-filter logic that converts structured beliefs to prose; worth verifying it doesn't drop meaningful content
- [general] `manual-tool-loop-vs-native-tool-use` — The module uses JSON-in-text tool calling rather than the Anthropic API's structured tool use; understanding tradeoffs here (portability across models vs. reliability of parsing) informs future refactoring

---

## Beliefs

- `ask-tool-loop-max-three-rounds` — The tool loop in `ask()` runs at most 3 iterations (`MAX_ITERATIONS = 3`); on the final round the prompt omits the tool definition to force a direct answer.
- `ask-dual-mode-makes-three-llm-calls` — Dual mode makes exactly 3 LLM calls in simple mode (1 TMS + 1 FTS RAG + 1 merge), short-circuiting to 2 if either path returns empty.
- `ask-llm-failure-returns-raw-beliefs` — When the LLM times out or raises, `ask()` falls back to returning the raw belief search results rather than raising an exception.
- `ask-tool-call-parsing-is-line-based-json` — Tool calls are detected by scanning each line of the LLM response for valid JSON with a `"tool"` key; multi-line JSON objects are not supported.
- `ask-sources-db-failure-silently-degrades` — If the `sources_db` SQLite file is missing or corrupt, `_search_source_chunks` catches the error and returns empty string, degrading to belief-only mode without user-visible errors.

