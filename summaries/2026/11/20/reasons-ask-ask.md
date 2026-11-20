---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: ask in reasons/ask.py

# `ask` — Answer a question over the belief network

## Purpose

`ask` is the top-level entry point of the `reasons` package for turning a natural-language question into an answer grounded in a **belief network** (a Truth Maintenance System stored in an FTS5 SQLite DB), optionally augmented with retrieved **source document chunks** (a separate FTS5 database) and **MCP tools**. It exists to be a single function that dispatches among four answer strategies:

1. **Dual** — run the belief-network path and the document-RAG path independently, then merge.
2. **No synthesis** — skip the LLM entirely and return raw belief search output.
3. **Simple** — one LLM call over beliefs (+ optional source chunks).
4. **Full (tool loop)** — iterative LLM loop that can request additional belief searches and call MCP tools before producing a final answer.

It sits downstream of `reasons.api.search` (retrieval) and `reasons.llm.invoke_model` (synthesis).

## Contract

**Preconditions**
- `question` is a non-empty string (not enforced; will be passed straight to FTS and into prompts).
- If `dual=True`, `sources_db` must be provided, else `ValueError("--dual requires --full-sources")` is raised.
- `db_path` must point at a belief-network SQLite DB that `api.search` understands.
- `sources_db`, if given, must be a valid `chunks_fts`-schema SQLite DB; `_search_source_chunks` catches `sqlite3.OperationalError`/`DatabaseError` and returns `""` if the query fails, so a malformed DB degrades to no documents.
- `model` must be one accepted by `invoke_model`.
- `mcp_servers` must be a list of `McpBridge`-like objects exposing `list_tools()`, `call_tool()`, `get_instructions()`.

**Postconditions**
- Returns a `str`. Never returns `None`.
- Returned string is either:
  - the LLM response (stripped), or
  - `NO_BELIEFS_MSG` (`"No matching beliefs found. Cannot answer from the belief network."`), or
  - raw belief context when the LLM fails but beliefs exist (via `_beliefs_or_no_match`), or
  - in `no_synth` mode, whatever `api.search` returned with the requested format.

**Invariants**
- Every LLM call is wrapped in a `try/except` for `TimeoutExpired` and generic `Exception`; those never propagate.
- The function writes progress to `stderr` at every major step (so putting it in a pipeline is programmatically safe for stdout).
- Nested/recursive calls from the `dual` path **never** pass `dual=True`, so they can't recurse infinitely.
- The full-mode loop always returns from inside; the trailing `return _beliefs_or_no_match(beliefs_context)` at the end is effectively unreachable and serves only as a safety net.

## Parameters

| Name | Type | Default | Meaning / Edge cases |
|---|---|---|---|
| `question` | `str` | — | The user question. Sent verbatim to FTS and to prompts. |
| `db_path` | `str` | `"reasons.db"` | Path to the belief-network FTS5 SQLite DB. |
| `timeout` | `int` | `300` | Per-LLM-call timeout in seconds. In `dual` mode it applies *per call* (TMS call, FTS call, merge call each get the full timeout). |
| `no_synth` | `bool` | `False` | If true, skip LLM entirely; return `api.search(...)` output. Overrides `simple`, `dual`, `sources_db`, etc. |
| `format` | `str\|None` | `None` | Only used when `no_synth=True`; defaults to `"compact"` via `format or "compact"`. |
| `model` | `str` | `"claude"` | Model identifier forwarded to `invoke_model`. |
| `simple` | `bool` | `False` | Single-pass LLM synthesis; disables the tool loop. Also used inside the `dual` path to configure the nested TMS call. |
| `sources_db` | `str\|None` | `None` | Path to `rag_fts.db`. Triggers appending `## Source Documents` excerpts after beliefs. **Required** for `dual`. |
| `natural` | `bool` | `False` | Strip belief IDs, `**Status:**`, justification metadata, etc. via `_strip_belief_metadata`, both from the prompt context and from `tool_history` results. |
| `dual` | `bool` | `False` | Run TMS and FTS retrieval independently and merge. Requires `sources_db` or raises. |
| `mcp_servers` | `list\|None` | `None` | Bridges that add extra tools to the prompt and extend `max_iters` from 3 to 5. |

Edge cases the type system does **not** enforce:
- `format` is only meaningful when `no_synth=True`; passing it otherwise is silently ignored.
- `sources_db=""` is falsy and thus treated as "no sources" even though it's a `str`.
- `mcp_servers=[]` is falsy — the code treats an empty list the same as `None`, so `max_iters` stays at 3.
- `dual=True` with `simple=True` still runs the TMS leg with `simple=True` and skips the tool loop; the docstring's "3 LLM calls" count assumes the caller understands this.

## Return Value

Returns a stripped answer string. Callers must be prepared to handle:

- **Normal LLM answer** — arbitrary text.
- **`NO_BELIEFS_MSG`** — returned when there are literally no matches, or after merge when both paths are empty, or on LLM failure with no beliefs.
- **Raw belief context (markdown/compact)** — returned by `no_synth=True`, or by `_beliefs_or_no_match(ctx)` when the LLM failed but beliefs exist. This is **structurally different** from an LLM answer: it contains `### <id>`, `**Status:**`, etc. Callers that expect prose need to check whether the return came from an LLM.
- **`"No relevant documents found for this question."`** — surfaces only via the dual path's internal `_fts_rag_answer`, but the dual merge logic filters it before it can reach the caller as the final answer unless the TMS answer is also empty.

There is **no signal** distinguishing "LLM failure fallback" from "LLM said this". A caller wanting to detect fallback must compare against `_beliefs_or_no_match` outputs or inspect stderr.

## Algorithm

1. **Dual short-circuit**
   - If `dual and not sources_db`: raise `ValueError`.
   - If `dual and sources_db`:
     - Recursively call `ask(question, ..., simple=simple, natural=natural, mcp_servers=mcp_servers)` — note: no `dual`, so the nested call takes a non-dual branch. No `sources_db` either — the TMS leg doesn't see source docs.
     - Call `_fts_rag_answer(...)` for the FTS-only leg.
     - Compute two emptiness predicates by **string matching**:
       - `tms_empty` if the answer equals `NO_BELIEFS_MSG` or contains `"don't have enough beliefs"` (the literal phrase the prompt tells the model to emit).
       - `fts_empty` if the FTS answer starts with `"No relevant documents"`.
     - If exactly one side is empty, return the other.
     - If both empty, return `NO_BELIEFS_MSG`.
     - Else call `_merge_answers(...)` (a third LLM call) and return its stripped output.

2. **`no_synth` short-circuit**
   - Return `api.search(question, db_path=db_path, format=format or "compact")`. No LLM touched.

3. **`simple` branch** (single LLM call)
   - `beliefs_context = api.search(..., format="markdown", depth=2)`.
   - Normalize `"No results found."` to `""`.
   - Optionally strip metadata via `_strip_belief_metadata` if `natural`.
   - If `sources_db`, retrieve `_search_source_chunks(...)` and append under `## Source Documents`.
   - If context is now empty, return `NO_BELIEFS_MSG`.
   - Build `SIMPLE_ASK_PROMPT` and call `invoke_model`.
   - On timeout/error, log and return `_beliefs_or_no_match(beliefs_context)` — i.e. raw beliefs if any, else `NO_BELIEFS_MSG`.
   - Return `response.strip()` otherwise.

4. **Full tool-loop branch**
   - `beliefs_context = api.search(..., format="markdown")` (no `depth` arg — different from simple branch).
   - Optional `natural` stripping.
   - `sources_suffix` captured once as a closure over `_full_context()`, so subsequent tool-requested belief searches replace `beliefs_context` but keep sources appended.
   - If `mcp_servers`: build tools section, instructions, and a `mcp_tool_map` of tool name → bridge. Bump `max_iters` to 5 (>= `MAX_ITERATIONS`).
   - Loop `for iteration in range(max_iters)`:
     - Context = `_full_context()`.
     - On the **last** iteration, use `build_final_prompt` (no tool options). Otherwise use `build_ask_prompt` (with tools + MCP instructions).
     - Call `invoke_model`. On timeout/error, log and `return _beliefs_or_no_match(ctx)`.
     - Parse `extract_tool_call(response)` — scans each line, parses JSON lines, returns the first dict with a `"tool"` key.
     - If no tool call → return `response.strip()`.
     - If `tool_name == "search_beliefs"` → re-run `api.search(query, format="markdown")`, append a `{tool_label, result}` entry to `tool_history` (metadata-stripped when `natural`), and replace `beliefs_context` when results are non-empty.
     - If `tool_name` matches an MCP tool → call `bridge.call_tool(tool_name, {args...})`, wrapping any exception into `"Error calling ..."`, append to `tool_history`.
     - If `tool_name` is unknown → return `response.strip()` (treat as final answer, don't retry).
     - If on the **last** iteration: do one extra `build_final_prompt` call, return its answer unless it too contained a tool call (then return `_beliefs_or_no_match(ctx)`).

5. **Fallback** — `return _beliefs_or_no_match(beliefs_context)`; unreachable in normal flow.

## Side Effects

- **Reads** from one or two SQLite databases via `api.search` and `_search_source_chunks` (opens/closes its own connection — no long-lived connection).
- **Invokes subprocesses** via `invoke_model` and (transitively) `subprocess.TimeoutExpired` is caught.
- **Calls MCP tools** — arbitrary remote side effects are possible through `bridge.call_tool`.
- **Writes to stderr** extensively (`Dual path: ...`, `Synthesizing (round X/Y)...`, `LLM timed out ...`, `Searching: ...`, `MCP tool: ...`). This is intentional: stdout stays clean.
- **No filesystem writes**, no in-process state mutation beyond local loop variables. `tool_history` is per-call.
- `mcp_servers` objects are not mutated by this function.

## Error Handling

- `ValueError("--dual requires --full-sources")` — the only explicit raise, and the only error type a caller should expect to catch on misuse.
- `subprocess.TimeoutExpired` on LLM call → logged, returns `_beliefs_or_no_match(context)`. Swallowed intentionally.
- Any other `Exception` from `invoke_model` → logged, same fallback. Swallowed.
- `sqlite3.OperationalError` / `sqlite3.DatabaseError` from `_search_source_chunks` → returns `""` (silent skip). **Errors from `api.search` are NOT caught** — a missing/corrupt `db_path` will propagate as a `sqlite3` error.
- MCP tool exceptions are caught and turned into `"Error calling {tool_name}: {e}"`, which goes into `tool_history` — the LLM still gets to reason about the failure.
- Errors from `extract_tool_call` (`json.JSONDecodeError`) are swallowed per-line; a malformed JSON line is simply skipped.
- **Behavioral coupling risk:** empty-detection is done via literal string matching against prompt instructions. If the prompt's refusal phrase changes, or the model paraphrases ("I don't have enough beliefs…" vs. "I do not have enough beliefs…"), the dual merge may consider a refusal to be a real answer.

## Usage Patterns

```python
from reasons.ask import ask

# Basic LLM-synthesized answer from beliefs
answer = ask("What causes X?")

# Single fast pass, no tool loop
answer = ask("What causes X?", simple=True)

# Include document excerpts
answer = ask("What causes X?", sources_db="rag_fts.db")

# Two independent retrievals merged into one answer
answer = ask("What causes X?", sources_db="rag_fts.db", dual=True)

# Skip the LLM entirely — raw belief dump
raw = ask("What causes X?", no_synth=True, format="markdown")

# Give the model external tools
from reasons.mcp import bridge
answer = ask("What causes X?", mcp_servers=[bridge])
```

Caller obligations:
- Check whether the return equals `"No matching beliefs found. …"` before formatting it as an answer.
- For CLI usage, forward stderr if you want the progress lines visible to the user.
- When using `natural=True`, expect only prose and no `[b1]`-style citations.
- When using `dual=True`, budget for 3–5 LLM calls × `timeout` seconds of wall time.

## Dependencies

- `reasons.api` (`search`) — primary retrieval.
- `reasons.llm` (`invoke_model`) — LLM synthesis for all synthesis paths.
- `subprocess` — only to catch `TimeoutExpired` re-raised by `invoke_model`.
- `sqlite3` — used indirectly via `search_source_chunks`.
- `json`, `re`, `sys` — utility.
- Module-local helpers: `build_simple_prompt`, `build_ask_prompt`, `build_final_prompt`, `extract_tool_call`, `_strip_belief_metadata`, `_search_source_chunks`, `_fts_rag_answer`, `_merge_answers`, `_beliefs_or_no_match`, `_build_tools_section`, `_build_mcp_instructions`, `MAX_ITERATIONS`, `NO_BELIEFS_MSG`.
- Indirect: MCP bridge implementations (external, duck-typed).

## Topics to Explore

- [file] `reasons/api.py` — `api.search` underpins every branch; understanding its `format`/`depth` options clarifies why `simple` uses `depth=2` and `dual`’s TMS leg doesn’t
- [function] `reasons/llm.py:invoke_model` — Determines what exceptions `ask` actually catches and whether timeouts are re-raised as `TimeoutExpired`
- [file] `tests/test_ask_mcp.py` — Shows how MCP tool calls are stubbed and how the tool loop is exercised end-to-end
- [general] `dual-mode-empty-detection` — String-matching the model’s refusal phrase is fragile; explore prompt/parse coupling invariants
- [general] `timeout-budgeting` — In `dual` mode `timeout` is per-call, not total; callers of `ask(..., dual=True)` may inadvertently block for minutes

## Beliefs

- `dual-requires-sources-db` — `dual=True` with `sources_db=None` raises `ValueError("--dual requires --full-sources")`.
- `no-synth-bypasses-llm` — When `no_synth=True`, `ask` returns `api.search(...)` directly and never calls `invoke_model`.
- `full-mode-uses-tool-loop` — Non-simple, non-dual, non-no_synth mode iterates up to `MAX_ITERATIONS` (3) or 5 with `mcp_servers`, feeding tool results back through `tool_history`.
- `llm-failures-return-beliefs` — LLM timeouts and exceptions are swallowed and return `_beliefs_or_no_match(ctx)` instead of propagating.
- `timeout-is-per-call` — The `timeout` parameter bounds each `invoke_model` call, not the total wall time of `ask`.
- `natural-strips-belief-metadata` — `natural=True` routes belief context and tool-history results through `_strip_belief_metadata` before they enter prompts.
- `empty-detection-is-string-match` — Empty-state detection in dual mode compares the returned string against `NO_BELIEFS_MSG`, `"don't have enough beliefs"`, and `"No relevant documents"` literally, so prompt changes can break emptiness detection.
