---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/ask.py

# `reasons/ask.py` — Question Answering Over a Belief Network

## 1. Purpose

This module is the **answer-synthesis engine** for the FTL Reasons project (a Truth Maintenance System). Given a natural-language question, it:

1. Searches the belief network (SQLite FTS5) for relevant beliefs.
2. Optionally augments context with full-text search over source-document chunks.
3. Invokes an LLM to synthesize a grounded answer from that context.

It owns the entire "ask a question → get an answer" pipeline, including the agentic tool-loop where the LLM can request additional belief searches, the dual-path (TMS + RAG) retrieval strategy, and MCP (Model Context Protocol) tool integration for external data sources.

The `reasons_lib/ask.py` file is a mirror copy (likely a vendored or library-published variant).

## 2. Key Components

### Constants / Prompt Templates

| Name | Role |
|------|------|
| `ASK_PROMPT` | Full prompt with tool definitions, used during tool-loop iterations (except the last). |
| `FINAL_ASK_PROMPT` | Prompt with **no** tool definition — forces the LLM to produce a final answer on the last iteration. |
| `SIMPLE_ASK_PROMPT` | Single-pass prompt, no tool loop. Used when `simple=True`. |
| `FTS_RAG_PROMPT` | Prompt for the FTS-RAG path (dual mode, answer B). |
| `MERGE_PROMPT` | Prompt that combines TMS answer + FTS answer into one (dual mode, step 3). |
| `_CITE_RULE` / `_CITE_RULE_NATURAL` | Toggle between "cite belief IDs in [brackets]" and "answer in plain natural language." |
| `MAX_ITERATIONS` (3) | Max tool-loop rounds. Bumped to 5 when MCP servers are present. |
| `NO_BELIEFS_MSG` | Canonical "I can't answer" string returned when search yields nothing. |

### Core Functions

| Function | Contract |
|----------|----------|
| **`ask(question, ...)`** | Main entry point. Returns answer text (str). Raises `ValueError` if `dual=True` but `sources_db` is None. |
| **`build_ask_prompt(...)`** | Renders `ASK_PROMPT` with tools section, tool history, and MCP instructions. Returns a formatted string. |
| **`build_final_prompt(...)`** | Renders `FINAL_ASK_PROMPT` (no tools). Used on last iteration to force a final answer. |
| **`build_simple_prompt(...)`** | Renders `SIMPLE_ASK_PROMPT`. No tool definitions at all. |
| **`extract_tool_call(text)`** | Scans LLM output line-by-line for a valid JSON object with a `"tool"` key. Returns the parsed dict or `None`. |
| **`search_source_chunks(query, sources_db, top_k=10)`** | Queries the `chunks_fts` FTS5 table. Returns `list[dict]` with keys `filename`, `section`, `text`, `cluster`. |
| **`_strip_belief_metadata(beliefs_context)`** | Strips structured TMS metadata (IDs, status, justifications, dependencies) from markdown-formatted beliefs, leaving plain prose. |
| **`_fts_rag_answer(...)`** | One-shot RAG: search source chunks → build `FTS_RAG_PROMPT` → call LLM → return answer string. |
| **`_merge_answers(...)`** | LLM call to merge two independent answers (TMS + FTS) into one coherent response. |
| **`_build_tools_section(mcp_servers)`** | Generates the tools section for the prompt, enumerating `search_beliefs` plus any MCP tools with their parameter schemas. |
| **`_build_mcp_instructions(mcp_servers)`** | Collects `get_instructions()` from each MCP bridge. |

### Helpers

- `_invoke_claude(prompt, timeout)` — thin backward-compat wrapper around `invoke_model`.
- `_search_source_chunks(question, sources_db)` — wraps `search_source_chunks` into a formatted markdown string; swallows SQLite errors.
- `_beliefs_or_no_match(beliefs_context)` — returns `NO_BELIEFS_MSG` if context is empty/"No results found.", otherwise passes through.

## 3. Patterns

- **Agentic Tool Loop**: The LLM is told it *may* respond with a single JSON tool-call line instead of a final answer. The loop (`for iteration in range(max_iters)`) feeds the result back into `tool_history` and re-prompts. On the **last** iteration the prompt switches to `FINAL_ASK_PROMPT` (no tool definition), forcing a terminal answer. This is a bounded ReAct-style loop.

- **Dual-Path Retrieval + Merge** (`dual=True`): Two independent retrieval strategies (structured TMS beliefs vs. raw document chunks) are run in sequence, then a third LLM call merges them. If one path is empty, the other is returned directly — no merge call.

- **Graceful Degradation**: Every LLM call site catches `subprocess.TimeoutExpired` and generic `Exception`. On failure, the raw belief context is returned rather than raising. The user still gets *something* grounded.

- **Template Rendering with Conditional Sections**: `mcp_instructions` and `tools_section` are injected as empty strings or populated text depending on whether MCP servers are connected. This avoids a separate prompt template per configuration.

- **Natural-Language Mode** (`natural=True`): A post-processing step (`_strip_belief_metadata`) removes all TMS-specific formatting (IDs, `**Status:**`, `**Justification:**`, dependency arrows) so the LLM sees plain prose. The cite rule also switches from "cite `[B-001]`" to "don't cite IDs."

- **MCP Bridge Pattern**: External tools are accessed through `bridge.list_tools()` / `bridge.call_tool(name, args)`. The tool name is mapped to a bridge instance in `mcp_tool_map` for dispatch.

## 4. Dependencies

### Imports (this file → others)

| Module | What's used |
|--------|-------------|
| `reasons.api` (`api`) | `api.search()` — FTS5 belief search; `api._STOP_WORDS` — for FTS query construction. |
| `reasons.llm` | `invoke_model(prompt, model=..., timeout=...)` — the sole LLM call mechanism. |
| `json`, `re`, `sqlite3`, `subprocess`, `sys` | Standard library. `sqlite3` is used directly for source-chunk FTS queries. `subprocess` is referenced for `TimeoutExpired` exception catching (the actual subprocess is inside `invoke_model`). |

### Imported By (others → this file)

| Module | Likely usage |
|--------|-------------|
| `reasons/api.py` | High-level API surface re-exports or delegates to `ask()`. |
| `reasons/cli.py` | CLI `ask` subcommand wiring. |
| `reasons_lib/ask.py` | Mirror/vendored copy. |
| `tests/test_ask.py`, `tests/test_ask_mcp.py`, `tests/test_search_sources.py` | Unit/integration tests for all modes. |

## 5. Flow (Control Path Through `ask()`)

```
ask(question, ...)
│
├─ dual AND sources_db?
│   ├─ YES →
│   │   1. answer_tms = ask(...) [recursive, non-dual]
│   │   2. answer_fts = _fts_rag_answer(...)
│   │   3. If one empty → return the other
│   │   4. Both non-empty → _merge_answers(...) → return merged
│   └─ NO → continue
│
├─ no_synth?
│   └─ YES → return api.search(question, format=...)  [raw text, no LLM]
│
├─ simple?
│   └─ YES →
│       1. beliefs_context = api.search(format="markdown", depth=2)
│       2. Optional: _strip_belief_metadata (if natural)
│       3. Optional: append _search_source_chunks (if sources_db)
│       4. build_simple_prompt → invoke_model → return
│       (single LLM call, no loop)
│
└─ DEFAULT (full tool-loop mode)
    1. beliefs_context = api.search(format="markdown")  [no depth limit]
    2. Optional: _strip_belief_metadata
    3. Optional: sources_suffix from _search_source_chunks
    4. Build tools_section / mcp_instructions if mcp_servers present
    5. for iteration in range(max_iters):   # 3, or 5 with MCP
    │     a. iteration == last?
    │     │   → build_final_prompt (no tools)
    │     └─ else
    │         → build_ask_prompt (with tools)
    │     b. invoke_model(...)
    │     c. extract_tool_call(response)
    │     │   → None?  return response  [final answer]
    │     │   → "search_beliefs"?
    │     │       → api.search(query) → append to tool_history
    │     │   → MCP tool name?
    │     │       → bridge.call_tool(...) → append to tool_history
    │     │   → unknown?  return response  [treat as final answer]
    │     └─ continue loop
    │
    └─ (if loop exhausts without final answer — should not happen
        because last iteration uses FINAL_ASK_PROMPT)
        → return _beliefs_or_no_match(beliefs_context)
```

**Data transformation pipeline**:
- Question → FTS5 query → belief rows → markdown context
- Context + question + tool_history → prompt string → LLM → response text
- Response → `extract_tool_call` → either JSON dict (loop) or final string (done)

## 6. Invariants

1. **`dual` requires `sources_db`**: Enforced at the top of `ask()` with `raise ValueError`. The two retrieval paths are only meaningful together.
2. **Bounded tool loop**: `MAX_ITERATIONS` (3) or `max(MAX_ITERATIONS, 5)` with MCP. The loop is hard-capped; no unbounded agent execution.
3. **Last iteration forces a final answer**: `build_final_prompt` omits the tools section entirely, so the LLM has no mechanism to request another tool call. If it *still* produces a tool-call JSON, `extract_tool_call` detects it and the raw context is returned instead.
4. **Grounding constraint** (prompt-level): The LLM is instructed to use *only* the provided beliefs and to respond with the exact string `"I don't have enough beliefs in the network to answer this question."` if insufficient. This is a soft invariant enforced by prompt, not code.
5. **Tool-call format**: Must be a single JSON line with a `"tool"` key. Multi-line or non-JSON responses are treated as final answers.
6. **`search_source_chunks` returns empty list** (not exception) when no words survive stop-word filtering or the FTS query matches nothing.

## 7. Error Handling

| Failure | Handling |
|---------|----------|
| LLM timeout (`subprocess.TimeoutExpired`) | Caught at every `invoke_model` site. Raw belief context returned via `_beliefs_or_no_match()`. Error printed to `sys.stderr`. |
| LLM / subprocess error (any `Exception`) | Same as timeout — graceful fallback to raw context. |
| SQLite errors in `_search_source_chunks` (`OperationalError`, `DatabaseError`) | Swallowed; returns `""`. The belief context is used without source-doc augmentation. |
| MCP `bridge.call_tool` exception | Caught inline; the error message is stored in `tool_history` as the result. The LLM sees the error text on the next iteration. |
| `extract_tool_call` JSON parse failure | Line is skipped. If no line yields a valid tool call, the response is treated as a final answer. |
| Empty / "No results found." belief context | `_beliefs_or_no_match()` converts to `NO_BELIEFS_MSG`. In `simple` mode, if context is entirely empty after all augmentation, `NO_BELIEFS_MSG` is returned without calling the LLM. |
| Unknown tool name in tool call | `ask()` returns the raw LLM response as-is (treated as a final answer). No retry. |

**Key observation**: This module *never raises* on LLM or retrieval failures. All errors are degraded into "here's the raw context" or "I can't answer." The only exception that propagates to the caller is the `ValueError` for `dual` without `sources_db`.

---

## Topics to Explore

- [function] `reasons/api.py:search` — The FTS5 belief search that feeds context into every ask path; understanding its `format`, `depth`, and scoring parameters is essential to debugging poor answer quality.
- [function] `reasons/llm.py:invoke_model` — The single LLM call boundary; its timeout semantics, model routing, and subprocess management determine the error-handling contract that `ask.py` relies on.
- [file] `tests/test_ask_mcp.py` — Exercises the MCP tool-loop path end-to-end; the best reference for understanding how `mcp_servers`, `mcp_tool_map`, and `_build_tools_section` interact.
- [file] `reasons/mcp_client.py` — Defines the `McpBridge` interface (`list_tools`, `call_tool`, `get_instructions`) that `ask.py` depends on for external tool dispatch.
- [general] `dual-path-retrieval-strategy` — The TMS + FTS-RAG + merge architecture is the most complex path; understanding when each path succeeds/fails and how the merge prompt resolves contradictions is critical for tuning answer quality.

---

## Beliefs

- `ask-requires-sources-db-for-dual-mode` — Calling `ask(..., dual=True)` without a `sources_db` path raises `ValueError` before any I/O or LLM call occurs.
- `tool-loop-is-bounded-by-max-iterations` — The agentic search loop in `ask()` executes at most 3 LLM calls (or 5 with MCP servers); the final iteration uses a prompt with no tool definition to force a terminal answer.
- `llm-failure-degrades-to-raw-context` — Every `invoke_model` call site in `ask.py` catches timeout and generic exceptions and returns the un-synthesized belief context rather than propagating the error to the caller.
- `natural-mode-strips-tms-metadata-before-llm` — When `natural=True`, `_strip_belief_metadata` removes all belief IDs, status markers, justification chains, and dependency edges from the context string before it reaches the LLM prompt.
- `extract-tool-call-maps-json-line-to-tool-dispatch` — `extract_tool_call` parses the first JSON line containing a `"tool"` key; the resulting dict's `"tool"` value is the dispatch key against `search_beliefs` and `mcp_tool_map` for routing.
