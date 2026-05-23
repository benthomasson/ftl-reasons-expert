# File: reasons_lib/ask.py

**Date:** 2026-05-11
**Time:** 13:03

## `reasons_lib/ask.py` — Natural Language Q&A Over a Belief Network

### 1. Purpose

This module is the **question-answering interface** for the TMS belief network. It takes a natural-language question, retrieves relevant beliefs via FTS5 full-text search, and optionally synthesizes a human-readable answer using an LLM. It owns the entire retrieval-augmented generation (RAG) pipeline for the `reasons ask` CLI command.

It supports three operating modes of increasing complexity:
- **No-synth** (`no_synth=True`): raw FTS5 search results, no LLM
- **Simple** (`simple=True`): single-pass LLM synthesis, no tool loop
- **Full** (default): multi-round LLM tool loop where the model can request additional `search_beliefs` calls or invoke external MCP tools

Plus a **dual-path** mode that runs TMS search and FTS document-chunk RAG independently, then merges the two answers in a third LLM call.

### 2. Key Components

**Prompt templates** (module-level constants):
- `ASK_PROMPT` — Main prompt with tool definitions and `{tool_history}` slot. Used in the iterative tool loop.
- `FINAL_ASK_PROMPT` — Same structure but without tool definitions, forcing the LLM to produce a final answer.
- `SIMPLE_ASK_PROMPT` — Minimal prompt with no tool definitions or history. Single-pass.
- `FTS_RAG_PROMPT` — Prompt for answering from raw source document chunks (not beliefs).
- `MERGE_PROMPT` — Prompt for merging two independently-produced answers (TMS + FTS).

**Prompt builders**:
- `build_simple_prompt()` — Formats `SIMPLE_ASK_PROMPT`.
- `build_ask_prompt()` — Formats `ASK_PROMPT` with tool history and optional MCP tools/instructions.
- `build_final_prompt()` — Formats `FINAL_ASK_PROMPT` (last-round, no tools).

**Tool extraction**:
- `extract_tool_call(text)` — Scans each line of LLM output for a JSON object with a `"tool"` key. Returns the first match or `None`. This is how the system detects the LLM requesting a follow-up search — it's a text-based tool-calling protocol, not native function calling.

**Natural-language mode**:
- `_strip_belief_metadata(beliefs_context)` — Strips `### ` headers, `**Status:**`, `**Justification:**`, etc., converting structured belief output into plain prose. Used when `natural=True`.

**Source document search**:
- `_search_source_chunks(question, sources_db, top_k=10)` — Queries a separate FTS5 index (`rag_fts.db`) of source document chunks. Tokenizes the question, strips stop words, builds an OR query, and returns formatted excerpts.

**MCP integration**:
- `_build_tools_section(mcp_servers)` — Generates the prompt's tool listing by iterating over connected `McpBridge` instances and their advertised tools.
- `_build_mcp_instructions(mcp_servers)` — Collects server-provided instructions for the prompt.

**Entry point**:
- `ask(question, ...)` — The main function. Orchestrates everything: mode selection, search, prompt construction, LLM invocation, tool-loop iteration, and dual-path merging.

### 3. Patterns

**Text-based tool calling**: Rather than using native LLM function-calling APIs, the model is instructed to emit a JSON line like `{"tool": "search_beliefs", "query": "..."}`. The `extract_tool_call` parser scans for this. This keeps the module LLM-agnostic — any model that can emit JSON works.

**Iterative refinement loop**: The main `ask()` path runs up to `MAX_ITERATIONS` (3, or 5 with MCP) rounds. Each round: build prompt → call LLM → check for tool call → execute tool → append to `tool_history` → repeat. On the final iteration, tools are stripped from the prompt to force an answer.

**Graceful degradation**: Every LLM call is wrapped in try/except for `TimeoutExpired` and generic `Exception`. On failure, the function returns the raw belief context (or a "no beliefs" message) rather than crashing. The system never loses the search results even if synthesis fails.

**Dual-path merge**: The `dual=True` mode runs `ask()` recursively for TMS, calls `_fts_rag_answer()` for documents, then merges via a third LLM call. It short-circuits if either path returns empty.

**Citation modes**: The `natural` flag controls two things simultaneously: (1) whether belief metadata is stripped from context, and (2) whether the prompt instructs the LLM to cite belief IDs in brackets or answer in plain language.

### 4. Dependencies

**Imports**:
- `api` (from `.`) — `api.search()` for FTS5 belief search, `api._STOP_WORDS` for query tokenization
- `llm.invoke_model` — Calls out to the configured LLM (Claude, etc.) via subprocess
- `sqlite3` — Direct DB access for source-chunk FTS queries
- `subprocess` — Only for catching `TimeoutExpired` from `invoke_model`

**Depended on by** (project-relevant):
- `reasons_lib/api.py` — Likely re-exports or delegates to `ask()`
- `reasons_lib/cli.py` — The `reasons ask` CLI command
- `tests/test_ask.py`, `tests/test_ask_mcp.py` — Test suites

### 5. Flow

The main `ask()` function follows this decision tree:

```
ask(question)
  ├─ dual=True?
  │    ├─ ask(question, dual=False)          → TMS answer
  │    ├─ _fts_rag_answer(question)          → FTS answer
  │    └─ _merge_answers(tms, fts)           → merged answer
  │
  ├─ no_synth=True?
  │    └─ api.search() → raw results
  │
  ├─ simple=True?
  │    ├─ api.search(format="markdown", depth=2)
  │    ├─ optionally strip metadata (natural)
  │    ├─ optionally append source chunks
  │    ├─ build_simple_prompt()
  │    └─ invoke_model() → answer
  │
  └─ (default: tool loop)
       ├─ api.search(format="markdown")
       ├─ optionally build MCP tools section
       └─ for iteration in 0..max_iters:
            ├─ build_ask_prompt() or build_final_prompt() (last iter)
            ├─ invoke_model()
            ├─ extract_tool_call()
            │    ├─ None → return answer
            │    ├─ "search_beliefs" → api.search(), update context
            │    ├─ MCP tool → bridge.call_tool(), append to history
            │    └─ unknown tool → return raw response
            └─ if last iteration with tool call → force final prompt
```

### 6. Invariants

- **`dual` requires `sources_db`**: Enforced with `ValueError` at the top of `ask()`.
- **Tool loop always terminates**: Hard-capped at `MAX_ITERATIONS` (3 or 5). The final iteration uses `build_final_prompt` which has no tool definitions, so even if the LLM emits a tool call, it's caught and the raw context is returned.
- **Stop-word filtering in `_search_source_chunks`**: Falls back to all words >1 char if every word is a stop word, ensuring the FTS query is never empty.
- **`natural` mode strips metadata before passing context to the LLM**: Both the initial context and tool-history results are stripped, maintaining consistency.

### 7. Error Handling

- **LLM timeouts/errors**: Caught per-call with `subprocess.TimeoutExpired` and `Exception`. Returns `_beliefs_or_no_match(ctx)` — the raw belief context if non-empty, otherwise `NO_BELIEFS_MSG`.
- **Source-chunk DB errors**: `_search_source_chunks` catches `sqlite3.OperationalError` and `DatabaseError`, returning empty string (silently skipped).
- **Invalid tool calls**: If the LLM emits a tool name that isn't `search_beliefs` or a registered MCP tool, the response is returned as-is (treated as a final answer).
- **MCP tool errors**: Caught with generic `Exception`, result becomes `"Error calling {tool_name}: {e}"` which is appended to tool history so the LLM sees the failure on the next round.

---

## Topics to Explore

- [file] `reasons_lib/llm.py` — How `invoke_model` dispatches to different LLM backends and handles timeouts
- [function] `reasons_lib/api.py:search` — The FTS5 search implementation that powers belief retrieval, including the `_STOP_WORDS` set and `format`/`depth` options
- [file] `reasons_lib/mcp_client.py` — The `McpBridge` class that `ask()` uses for external tool integration
- [file] `tests/test_ask.py` — Test cases showing how the tool loop, natural mode, and dual-path mode are exercised
- [general] `text-based-tool-calling-vs-native` — Whether migrating to native function calling (e.g., Claude tool_use) would improve reliability of tool extraction

## Beliefs

- `ask-tool-loop-max-3-rounds` — The default tool loop runs at most 3 iterations (`MAX_ITERATIONS`), extended to 5 when MCP servers are connected
- `ask-extract-tool-call-is-line-based-json` — Tool calls are detected by scanning each line of the LLM response for a JSON object containing a `"tool"` key; this is not native function calling
- `ask-llm-failure-returns-raw-context` — When the LLM times out or raises an exception, `ask()` returns the raw belief search results rather than propagating the error
- `ask-dual-mode-makes-three-llm-calls` — Dual-path mode invokes the LLM at least 3 times: once for TMS, once for FTS RAG, once for merging (unless one path is empty)
- `ask-natural-strips-metadata-and-citations` — The `natural=True` flag both strips structured belief metadata from context and instructs the LLM not to cite belief IDs

