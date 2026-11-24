---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: _fts_rag_answer in reasons/ask.py

# `_fts_rag_answer` — explanation

## Purpose

`_fts_rag_answer` is the "Answer B" producer for the **dual retrieval** path of `ask()`. It runs a classic naive RAG loop over source-document chunks: retrieve via FTS5, synthesize via an LLM, return the text.

It exists to be paired with the belief-network ("Answer A") path so `_merge_answers` can combine structured belief knowledge with passages from the underlying source documents. It is the FTS-only retrieval pipeline without any TMS/tool-loop machinery.

## Contract

**Preconditions**
- `sources_db` points at a valid SQLite database exposing `chunks_fts` (FTS5 virtual table) and `chunks` tables (populated by `rag_fts.db`). This is not enforced by the type system; a wrong path typically results in a silently created empty DB and a `no such table` error that `_search_source_chunks` swallows.
- `question` is a non-empty string.
- `model` and `timeout` are acceptable to `invoke_model`.

**Postconditions**
- Returns a non-empty string. Either the sentinel `"No relevant documents found for this question."` or the LLM response with leading/trailing whitespace stripped.
- Makes at most one LLM call.
- Does not mutate `sources_db` or any caller state.

**Invariants**
- If and only if `_search_source_chunks(...)` returns a falsy string, the sentinel is returned and no LLM call is made.
- The LLM is invoked with `FTS_RAG_PROMPT` filled in; no other prompt variant is used here.

## Parameters

| Name | Type (expected) | Meaning | Edge cases |
|---|---|---|---|
| `question` | `str` | Natural-language question; used both as FTS query input and inserted into the prompt. | Empty/whitespace → `search_source_chunks` filters out stop words and short tokens; if nothing survives it returns `[]` → `_search_source_chunks` returns `""` → sentinel returned. |
| `sources_db` | `str` (path) | Path to SQLite FTS5 database. | Nonexistent path: `sqlite3.connect` creates an empty file; subsequent `MATCH` on `chunks_fts` raises `OperationalError`, which `_search_source_chunks` catches and maps to `""`. |
| `model` | `str` | Model identifier for `invoke_model`. Default `"claude"`. | Invalid model name → whatever `invoke_model` throws propagates (see Error Handling). |
| `timeout` | `int` seconds | Per-LLM-call timeout, passed straight through. Default 300. | Zero/negative → `invoke_model`/`subprocess` may raise; not guarded here. |

## Return Value

- `"No relevant documents found for this question."` — literal early return when retrieval yields nothing (no rows, DB error, or empty token list after stop-word filtering).
- `invoke_model(prompt, model=model, timeout=timeout).strip()` — LLM output with outer whitespace removed. If the LLM returns whitespace-only text, the caller receives `""` (an edge case that the dual-mode caller’s `startswith("No relevant documents")` check does **not** catch, so it will treat `""` as a valid FTS answer and proceed to merge).

Callers must handle:
- A possible `""` (whitespace-only LLM output).
- Exceptions from `invoke_model` (this function does not catch them).

## Algorithm

1. Call `_search_source_chunks(question, sources_db)` — described below.
2. If the returned context is falsy, return the sentinel string immediately.
3. Build the prompt via `FTS_RAG_PROMPT.format(context=context, question=question)`.
4. Call `invoke_model(prompt, model=model, timeout=timeout)`.
5. Return its `.strip()` output.

The retrieval step (`_search_source_chunks` → `search_source_chunks`):
- Tokenizes `question` with `re.findall(r'\w+', ...)`, drops `_STOP_WORDS` and single-character tokens; falls back to all tokens with length > 1 if the stop-word filter empties the list; returns `[]` if there’s nothing usable.
- Builds an OR-of-quoted-terms FTS5 query.
- Runs `SELECT ... FROM chunks_fts JOIN chunks ... MATCH ? ORDER BY rank LIMIT 10` on a fresh connection (`top_k=10`).
- Formats each row as `### [i] filename > section\n\ntext` and joins with `\n\n---\n\n`.
- Swallows `sqlite3.OperationalError` / `sqlite3.DatabaseError` and returns `""`.

## Side Effects

- Opens and closes its own `sqlite3` connection inside `search_source_chunks` (read-only in spirit; no writes).
- Makes one external LLM call via `invoke_model` (typically a subprocess invocation of the `claude` CLI in this codebase).
- Performs no `print()` / logging in this function or in `_search_source_chunks`.
- No global state mutated.

## Error Handling

- **SQLite / retrieval errors** (`OperationalError`, `DatabaseError`): swallowed in `_search_source_chunks`, mapped to `""`, which becomes the sentinel return. Missing-table and malformed-FTS-query errors therefore never reach the caller.
- **Other sqlite exceptions** (`ProgrammingError`, connection-open failures that aren’t `DatabaseError`) are **not** caught by `_search_source_chunks` and will propagate through this function.
- **LLM errors** (`subprocess.TimeoutExpired`, arbitrary `Exception` from `invoke_model`): **not caught here**. This is a deliberate asymmetry — the surrounding TMS paths in `ask()` do catch these and fall back to `_beliefs_or_no_match(...)`, but the dual-path FTS branch has no such guard, so a timeout during FTS RAG will surface as an exception from `ask()` rather than a graceful "no docs" answer.

## Usage Patterns

- Sole caller: `ask()` inside the `if dual and sources_db:` branch, after the recursive TMS call. It runs unconditionally there:
  ```python
  answer_fts = _fts_rag_answer(question, sources_db, model=model, timeout=timeout)
  ```
- Caller obligations:
  - Ensure `sources_db` is provided (guarded up front by `raise ValueError("--dual requires --full-sources")`).
  - Inspect the returned string with `answer_fts.strip().startswith("No relevant documents")` to decide whether to short-circuit to the TMS answer, return `NO_BELIEFS_MSG`, or proceed to `_merge_answers`.
  - Beware that an empty-string return from the LLM is treated as a non-empty answer by that check.
- Not used for the non-dual `sources_db` path; those branches call `_search_source_chunks(question, sources_db)` directly and append the results to the belief context before prompting.

## Dependencies

- `FTS_RAG_PROMPT` (same module) — the answer template that instructs the model to cite filenames in `[...]` and admit gaps.
- `_search_source_chunks` → `search_source_chunks` (same module) — FTS5 retrieval + formatting; imports `_STOP_WORDS` from `.api` and uses `sqlite3`.
- `invoke_model` (from `.llm`) — the actual LLM transport; may raise `subprocess.TimeoutExpired` on timeout per patterns seen elsewhere in this file.
- Indirect: `MERGE_PROMPT` is *not* touched here, but its output is what consumes this function’s answer.

**Implicit assumptions not enforced by the type system**
- `sources_db` is a filesystem path, not an in-memory `sqlite3.Connection` or `Path` (the code passes it directly to `sqlite3.connect`).
- The database schema uses `chunks_fts.rowid` aligned with `chunks.id` (the JOIN assumes this).
- `invoke_model` returns a `str` (`.strip()` is called directly on the return value with no cast).

## Topics to Explore

- [function] `reasons/ask.py:_search_source_chunks` — The upstream retrieval/formatting adapter that determines what `context` will contain and how DB errors become empty results.
- [function] `reasons/ask.py:search_source_chunks` — The raw FTS5 query with stop-word filtering and `top_k` ranking; useful to understand query construction edge cases.
- [function] `reasons/llm.py:invoke_model` — What it actually shells out to, what exceptions it raises on timeout vs. process failure, and whether it can return an empty string.
- [function] `reasons/ask.py:ask` — The dual-path orchestration (`tms_empty` / `fts_empty` detection, merge short-circuits) that consumes this function’s return value.
- [file] `tests/test_ask.py` — Existing tests exercising the dual path; useful to see which sentinel/edge-case behaviors are already asserted.

## Beliefs

- `fts-rag-is-dual-path-only` — `_fts_rag_answer` is only invoked from `ask()` when `dual=True` and `sources_db` is provided; other `sources_db` usages call `_search_source_chunks` directly.
- `fts-rag-requires-sources-db` — `_fts_rag_answer` depends on `sources_db` exposing `chunks_fts`/`chunks`; invalid paths yield the sentinel because `_search_source_chunks` swallows `sqlite3.OperationalError`/`DatabaseError`.
- `fts-rag-propagates-llm-errors` — `_fts_rag_answer` does **not** catch `subprocess.TimeoutExpired` or generic exceptions from `invoke_model`, unlike the TMS paths in `ask()` that fall back to `_beliefs_or_no_match`.
- `fts-rag-produces-cited-llm-answer` — It returns an LLM-synthesized answer intended to cite source document filenames in `[filename]` form per `FTS_RAG_PROMPT`.
- `fts-rag-empty-result-sentinel` — When retrieval returns nothing, `_fts_rag_answer` returns the exact string `"No relevant documents found for this question."`, which the dual-path caller detects via `startswith("No relevant documents")`.
