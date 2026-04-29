# File: reasons_lib/ask.py

**Date:** 2026-04-29
**Time:** 17:12



# `reasons_lib/ask.py` — Natural Language Q&A Over the Belief Network

## Purpose

This module implements a **natural language question-answering interface** over the TMS belief store. It bridges the gap between raw FTS5 search results (which return matching beliefs) and synthesized, human-readable answers. It does this by orchestrating a tool-use loop: it searches for relevant beliefs, hands them to Claude via the `claude` CLI, and lets the LLM request additional searches before composing a final answer.

It owns exactly one responsibility: turning a question string into an answer string, using the belief database as its knowledge source.

## Key Components

### Constants

- **`ASK_PROMPT`** — The system prompt template. Instructs the LLM to answer from belief matches, use `[bracket]` citations for belief IDs, and optionally call a `search_beliefs` tool by emitting a single JSON line. The template has three interpolation slots: `{question}`, `{beliefs_context}`, and `{tool_history}`.

- **`FINAL_TURN_INSTRUCTION`** — Appended to the prompt on the last iteration to force the LLM to stop issuing tool calls and produce a final answer.

- **`MAX_ITERATIONS = 3`** — Hard cap on the tool-use loop. The LLM gets at most 3 rounds: the initial response plus up to 2 additional searches.

### Functions

**`extract_tool_call(text) → dict | None`** — Parses LLM output line-by-line looking for a JSON object with a `"tool"` key. Returns the first match or `None`. This is the hand-rolled tool-use protocol — no function-calling API, just convention over raw text.

**`build_ask_prompt(question, beliefs_context, tool_history=None) → str`** — Assembles the full prompt by formatting `ASK_PROMPT` and appending any accumulated tool-call results as `## Additional search results` sections separated by `---`.

**`_invoke_claude(prompt, timeout=300) → str`** — Shells out to the `claude` CLI via `subprocess.run` with the prompt on stdin. Three failure modes are explicitly surfaced: `FileNotFoundError` if `claude` isn't installed, `RuntimeError` on non-zero exit, `TimeoutExpired` on timeout. Notably strips the `CLAUDECODE` environment variable to prevent recursive invocation when running inside Claude Code.

**`ask(question, db_path, timeout, no_synth, format) → str`** — The main entry point. Two modes:
- **`no_synth=True`**: bypasses the LLM entirely, returns raw FTS5 search results (uses the `format` parameter, defaulting to `"compact"`).
- **`no_synth=False`** (default): runs the full search → synthesize → tool-loop pipeline.

## Patterns

### Hand-Rolled Tool Use
Rather than using Claude's native tool-use API, this module implements a **text-based tool protocol**. The LLM is instructed to emit a JSON line like `{"tool": "search_beliefs", "query": "..."}`, and `extract_tool_call` parses it out. This works because the module calls `claude -p` (the CLI's pipe mode), which doesn't support the tool-use API. It's a pragmatic choice — the CLI is the simplest integration path.

### Graceful Degradation
Every failure in the LLM path falls back to returning raw `beliefs_context` (the initial FTS5 search results). Timeout, exception, or exhausted iterations — the user always gets *something*.

### Bounded Iteration
The tool loop is capped at `MAX_ITERATIONS = 3` with a `FINAL_TURN_INSTRUCTION` forcing termination on the last round. This prevents runaway LLM loops and keeps cost/latency predictable.

### Environment Scrubbing
`_invoke_claude` strips `CLAUDECODE` from the environment. This is a re-entrancy guard — without it, running `reasons ask` inside a Claude Code session would cause the subprocess to think it's *also* inside Claude Code.

## Dependencies

**Inbound (imports):**
- `json`, `os`, `shutil`, `subprocess`, `sys` — standard library
- `api` (from `reasons_lib`) — provides `api.search()`, the FTS5 search interface into the belief database

**Outbound (imported by):**
- `reasons_lib/cli.py` — wires the `ask` subcommand to `ask()`
- `reasons_lib/api.py` — re-exports or references it
- `tests/test_ask.py` — unit tests

## Flow

```
ask(question)
 │
 ├── no_synth=True? ──→ api.search(question, format) ──→ return raw results
 │
 ├── api.search(question, format="markdown") ──→ beliefs_context
 │
 └── loop (up to MAX_ITERATIONS):
      │
      ├── build_ask_prompt(question, beliefs_context, tool_history)
      │
      ├── (last iteration? append FINAL_TURN_INSTRUCTION)
      │
      ├── _invoke_claude(prompt) ──→ response
      │     │
      │     └── on error ──→ return beliefs_context (fallback)
      │
      ├── extract_tool_call(response)
      │     │
      │     ├── None or last iteration ──→ return response (final answer)
      │     │
      │     ├── tool == "search_beliefs" ──→ api.search(query)
      │     │     └── append to tool_history, continue loop
      │     │
      │     └── unknown tool ──→ return response as-is
      │
      └── (loop exhausted) ──→ return beliefs_context
```

## Invariants

1. **The function always returns a string.** Every code path in `ask()` returns either the LLM response, the raw search results, or the fallback `beliefs_context`. It never raises to the caller.
2. **Tool history is append-only and monotonically grows.** Each iteration may add one entry; entries are never removed or modified.
3. **The LLM sees all prior search results.** `build_ask_prompt` includes the full `tool_history` every round, giving the LLM cumulative context.
4. **At most `MAX_ITERATIONS` LLM calls are made.** The loop is bounded; the last iteration always forces a final answer.
5. **`claude` CLI must be in PATH for synthesis.** The `no_synth` path works without it; the synthesis path raises `FileNotFoundError` (caught and fallen back from in `ask()`).

## Error Handling

Errors are **caught and degraded, never propagated** from `ask()`:

| Error | Source | Handling |
|---|---|---|
| `subprocess.TimeoutExpired` | `_invoke_claude` | Logged to stderr, returns raw `beliefs_context` |
| `FileNotFoundError` | `_invoke_claude` (no `claude` CLI) | Caught by the generic `except Exception`, returns `beliefs_context` |
| `RuntimeError` | `_invoke_claude` (non-zero exit) | Same — caught, logged, fallback |
| `json.JSONDecodeError` | `extract_tool_call` | Silently skipped (line wasn't valid JSON) |

The design philosophy is clear: LLM synthesis is best-effort. If it fails, you still get the raw search results. The only output channel for errors is `sys.stderr`.

## Topics to Explore

- [file] `reasons_lib/api.py` — Implements `api.search()`, the FTS5 search that feeds both the initial context and tool-call results
- [file] `tests/test_ask.py` — Shows how the tool loop is tested, likely with mocked `_invoke_claude` to simulate multi-round conversations
- [function] `reasons_lib/cli.py:ask` — The CLI subcommand that wires user arguments (`--no-synth`, `--format`, `--timeout`) to this module
- [general] `claude-cli-pipe-mode` — Understanding `claude -p` behavior and its limitations vs. the API (no native tool use, no streaming)
- [file] `reasons_lib/storage.py` — The SQLite/FTS5 layer that `api.search` queries against

## Beliefs

- `ask-always-returns-string` — `ask()` returns a string on every code path; it never raises an exception to the caller
- `ask-tool-loop-capped-at-three` — The LLM synthesis loop runs at most `MAX_ITERATIONS` (3) rounds, with the final round forced to produce an answer
- `ask-falls-back-to-raw-search` — When LLM synthesis fails for any reason (timeout, missing CLI, non-zero exit), `ask()` returns the raw FTS5 search results as fallback
- `ask-strips-claudecode-env` — `_invoke_claude` removes the `CLAUDECODE` environment variable to prevent recursive invocation when running inside Claude Code
- `ask-no-synth-bypasses-llm` — When `no_synth=True`, `ask()` returns raw `api.search()` results without invoking the Claude CLI

