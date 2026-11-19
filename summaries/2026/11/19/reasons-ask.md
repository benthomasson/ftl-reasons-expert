---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/ask.py

# `reasons/ask.py` — Question Answering over a Belief Network

## Purpose

This module is the top-level **question-answering entry point** for the `reasons` belief network (a Truth Maintenance System). It:

1. Runs an FTS5 search over beliefs (via `api.search`) to get candidate context.
2. Optionally signals retrieved document chunks via a separate FTS5 index over source documents.
3. Synthesizes a natural-language answer using an LLM, optionally in a **tool loop** where the model can request additional belief searches, or call external MCP tools.
4. Supports multiple retrieval strategies (`simple`, `dual`, `no_synth`) and a `natural` presentation mode.

Its output is plain text (`str`), with a well-known sentinel message (`NO_BELIEFS_MSG`) for "I have nothing useful."

---

## Key Components

### Prompt templates (constants)

| Constant | Role |
|---|---|
| `ASK_PROMPT` | Tool-loop round prompt. Includes `tools_section`, `cite_rule`, `mcp_instructions`, `beliefs_context`, `tool_history`. |
| `FINAL_ASK_PROMPT` | Last-round prompt, **no tool definitions** — forces a written answer. |
| `SIMPLE_ASK_PROMPT` | Single-pass, no tool definitions. |
| `FTS_RAG_PROMPT` | Answers from source-document excerpts with `[filename]` citations. |
| `MERGE_PROMPT` | Merges TMS answer (A) with FTS RAG answer (B) in dual mode. |
| `_CITE_RULE` / `_CITE_RULE_NATURAL` | Cite belief IDs vs. plain prose. |
| `_DEFAULT_TOOLS_SECTION` | Fallback tool description when no MCP servers are attached. |

All prompts contain the exact fallback phrase *"I don't have enough beliefs in the network to answer this question."* — this is a **contract**, because dual mode detects emptiness by substring.

### Helper functions

- **`build_simple_prompt`, `build_ask_prompt`, `build_final_prompt`** — pure template formatters. `build_ask_prompt` / `build_final_prompt` collapse `tool_history` entries (`{'tool_label', 'result'}`) into a `## Additional search results` block separated by `---`.
- **`extract_tool_call(text)`** — the wire protocol. Splits on `\n`, keeps lines starting with `{`, tries `json.loads`, returns the first dict that has a `"tool"` key. Returns `None` otherwise. This is how the LLM "calls a tool."
- **`_strip_belief_metadata(beliefs_context)`** — converts the structured markdown belief context into prose by dropping `### ` headers, `**Status:**`, `**Justification:**`, `**Depends on:**`, `- **id** (IN|OUT):` bullet lines, and `---` separators, then collapses 3+ newlines to 2. Used when `natural=True`.
- **`_invoke_claude(prompt, timeout=300)`** — legacy backward-compat wrapper around `invoke_model(prompt, model="claude")`. Not used inside `ask` anymore.
- **`search_source_chunks(query, sources_db, top_k=10)`** — builds an OR-of-quoted-words FTS5 query from non-stopword tokens (falls back to any word >1 char), joins `chunks_fts` to `chunks`, returns dicts with `filename, section, text, cluster`. Raises `sqlite3` errors upward.
- **`_search_source_chunks(question, sources_db, top_k=10)`** — same, but formatted as `### [i] filename > section` blocks separated by `---`; swallows `sqlite3.OperationalError`/`DatabaseError` and returns `""`.
- **`_fts_rag_answer(question, sources_db, model, timeout)`** — runs source-document RAG end-to-end. Returns `"No relevant documents found for this question."` when nothing is retrieved (another sentinel checked in `dual`).
- **`_merge_answers(...)`** — third LLM call in dual mode.
- **`_beliefs_or_no_match(beliefs_context)`** — returns `NO_BELIEFS_MSG` if context is empty or equals `"No results found."`, otherwise the context.
- **`_build_tools_section(mcp_servers)`** — emits one JSON example line per `bridge.list_tools()` tool, deriving parameter placeholders from `input_schema.properties`, plus a `# description` comment.
- **`_build_mcp_instructions(mcp_servers)`** — concatenates each bridge's `get_instructions()`.

### `ask(...)` — the main dispatcher

`ask(question, db_path="reasons.db", timeout=300, no_synth=False, format=None, model="claude", simple=False, sources_db=None, natural=False, dual=False, mcp_servers=None)`

Modes are selected in this **priority order**:

1. **`dual`** — requires `sources_db` (raises `ValueError` otherwise). Runs TMS `ask` (recursive), then `_fts_rag_answer`, then decides:
   - TMS empty & FTS non-empty → FTS answer.
   - FTS empty & TMS non-empty → TMS answer.
   - Both empty → `NO_BELIEFS_MSG`.
   - Both non-empty → `_merge_answers`.
2. **`no_synth`** — calls `api.search` directly with `format` (default `"compact"`).
3. **`simple`** — one LLM call. Context from `api.search(..., depth=2)`, optionally stripped (`natural`) and appended with source chunks. If still empty → `NO_BELIEFS_MSG`.
4. **Default tool-loop** — up to `MAX_ITERATIONS` (3) rounds, or `max(MAX_ITERATIONS, 5)` if `mcp_servers` are present. Each round: build prompt (full → final on last round), call `invoke_model`, parse `extract_tool_call`. If a tool is called (`search_beliefs` or an MCP tool), record in `tool_history` and loop. Otherwise return the response.

---

## Patterns

- **Strategy by parameter flags** — the single `ask` function dispatches on `dual`, `no_synth`, `simple`. Nothing is subclassed; each strategy is inline.
- **Text-based tool calling** — a lightweight, model-agnostic JSON-lines protocol instead of native function-calling APIs. `extract_tool_call` is the parser.
- **Prompt templating via `.format`** — all prompts are module-level strings with `{…}` placeholders; `mcp_instructions` is post-formatted into a `## Data Source Instructions` block when present.
- **Graceful degradation** — every LLM call is wrapped in `try/except`; on error it falls back to raw belief context.
- **Staged synthesis** — belief tool-loop ends with a "final" prompt that omits tool schemas, preventing an infinite tool-call loop.
- **Mode-gated short-circuits in `dual`** — checks emptiness sentinels so merging only happens when both sides contributed.

---

## Dependencies

**Imports:**
- Stdlib: `json`, `re`, `sqlite3`, `subprocess`, `sys`.
- `from . import api` — used for `api.search(question, db_path=..., format="markdown"|"compact", depth=2)`.
- `from .llm import invoke_model` — LLM call boundary. Raises `subprocess.TimeoutExpired` on timeout.
- Lazy: `from .api import _STOP_WORDS` inside `search_source_chunks`.

**Imported by:**
- `reasons/api.py` — likely re-exports or wraps `ask`.
- `reasons/cli.py` — the `ask` CLI subcommand.
- `reasons_lib/ask.py` — the library mirror package.
- `tests/test_ask.py`, `tests/test_ask_mcp.py`, `tests/test_search_sources.py`.

---

## Flow (tool loop, default mode)

```
ask()
 ├─ dual?         → TMS→FTS→merge/short-circuit
 ├─ no_synth?     → api.search() directly
 ├─ simple?       → api.search() + build_simple_prompt → invoke_model → return
 └─ else:
      beliefs_context = api.search(format="markdown")
      sources_suffix  = optional _search_source_chunks()
      for iteration in range(max_iters):
        ctx   = beliefs_context (+ sources_suffix)
        prompt = FINAL_ASK_PROMPT  if last round
               = ASK_PROMPT         otherwise
        response = invoke_model(prompt)
        call = extract_tool_call(response)
        if call is None: return response.strip()
        if call.tool == "search_beliefs":
              result = api.search(result)
              tool_history += {label, result}
              if meaningful: beliefs_context = result   ← update only on non-empty
        elif call.tool in mcp_tool_map:
              result = bridge.call_tool(...)
              tool_history += {label, result}
        else: return response.strip()
      # Last-round fallback: if a tool was still emitted, force a final answer.
```

`_full_context()` is the single source for what the model sees: `beliefs_context + sources_suffix` — ensuring new search results *replace* the belief portion, not append, while source docs persist.

---

## Invariants

- **`dual` requires `sources_db`** — enforced with `ValueError("--dual requires --full-sources")` as the very first check.
- **Final round carries no tool schema** — `build_final_prompt` uses `FINAL_ASK_PROMPT` which has no tool block. If the model still returns a tool call, fall back to `_beliefs_or_no_match(ctx)` rather than loop.
- **Tool result only replaces `beliefs_context` when meaningful** — `if result and result.strip() != "No results found."` gates the update; empty searches still appear in `tool_history`.
- **`extract_tool_call` accepts only `{…}` lines with a `"tool"` key** — mixed prose is ignored; multiple calls → first one wins.
- **Sentinel coupling in dual mode** — `tms_empty` matches `NO_BELIEFS_MSG` *or* the substring `"don't have enough beliefs"`; `fts_empty` matches `answer_fts.startswith("No relevant documents")`. Changing prompt strings or `_fts_rag_answer`'s sentinel silently breaks dual mode.
- **`natural=True` produces no belief IDs** — enforced by both `_CITE_RULE_NATURAL` and `_strip_belief_metadata` scrubbing headers and bullet lines.
- **MCP tool calls cannot bypass the loop cap** — `max_iters = max(MAX_ITERATIONS, 5)` still terminates.

---

## Error Handling

| Failure | Handling |
|---|---|
| LLM timeout (`subprocess.TimeoutExpired`) | Print to stderr; return `_beliefs_or_no_match(ctx)` — raw context fallback. |
| Generic `Exception` from LLM | Print to stderr; return raw context fallback. |
| SQLite error in source-chunk lookup (`_search_source_chunks`) | Caught (`OperationalError`, `DatabaseError`) → `""`. Note `search_source_chunks` itself propagates. |
| MCP tool exception | Caught and converted to `f"Error calling {tool_name}: {e}"` recorded in `tool_history` — the loop continues. |
| Unknown tool name in response | Return the raw response text. |
| `dual` without `sources_db` | `ValueError` — programmer error, not swallowed. |
| Empty belief set | `NO_BELIEFS_MSG` (either via `_beliefs_or_no_match` or explicit check in `simple` mode). |

A quirk to note: the outer `return _beliefs_or_no_match(beliefs_context)` at the end of the tool loop is effectively **unreachable** — every path through the last iteration either returns a response or returns from the final-synthesis block. It exists as a defensive terminator.

---

## Topics to Explore

- [function] `reasons/api.py:search` — Understand `format="markdown"|"compact"` and `depth=2`; the entire tool-loop's belief context format depends on it.
- [file] `reasons/llm.py` — `invoke_model` defines the model routing, timeout semantics (`subprocess.TimeoutExpired`), and the actual subprocess call boundary.
- [file] `reasons/mcp_client.py` — `McpBridge.list_tools`, `get_instructions`, `call_tool` define the MCP tool contract that `_build_tools_section` and the loop rely on.
- [general] `tool-calling-json-lines-protocol` — Design rationale for text-based vs native function calling, and how `extract_tool_call` can fail silently on pretty-printed JSON.
- [file] `tests/test_ask.py` — Exercises each mode (`simple`, `dual`, `no_synth`, timeout fallback), pinning down the expected sentinel strings and iteration behavior.

## Beliefs

- `ask-requires-sources-db-for-dual-mode` — `ask` raises `ValueError` at the top if `dual=True` and `sources_db is None`.
- `extract-tool-call-implements-json-line-protocol` — `extract_tool_call` parses the first line beginning with `{` that loads as JSON and contains a `"tool"` key.
- `ask-prevents-tool-calls-in-final-round` — On the last iteration, `build_final_prompt` (no tool schema) is used, and any returned tool call falls back to raw context.
- `tool-search-results-replace-beliefs-context-when-non-empty` — A `search_beliefs` result only overwrites `beliefs_context` when it is non-empty and not `"No results found."`.
- `dual-mode-detects-emptiness-via-prompt-substrings` — Dual mode classifies an empty TMS answer via the exact `NO_BELIEFS_MSG` string or the substring `"don't have enough beliefs"`, coupling it to prompt wording.
- `ask-falls-back-to-raw-context-on-llm-errors` — Both `subprocess.TimeoutExpired` and generic exceptions from `invoke_model` return `_beliefs_or_no_match(ctx)` instead of raising.
