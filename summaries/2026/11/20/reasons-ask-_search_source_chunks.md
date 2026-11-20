---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: _search_source_chunks in reasons/ask.py

# `_search_source_chunks` — Explanation

## Purpose

`_search_source_chunks` is a **presentation-layer adapter** between the structured FTS5 search helper and the LLM prompt builders. It exists so that callers in `ask()` and `_fts_rag_answer()` can drop source-document context into a prompt as a pre-formatted markdown string, without caring about SQLite plumbing or per-row formatting.

- The sibling `search_source_chunks` (no underscore) does the actual FTS5 query and returns `list[dict]`.
- `_search_source_chunks` iterates over that list, formats each chunk into a markdown section with a bracketed index `[i]`, and joins them with horizontal rules.

It is used in three places:

- Simple path of `ask()` → appended under `"## Source Documents\n\n"`.
- Non-simple path of `ask()` → appended under `"## Source Documents\n\n"`.
- `_fts_rag_answer()` → fed into `FTS_RAG_PROMPT` as `{context}`.

## Contract

**Preconditions** (not enforced by types):
- `sources_db` must be a truthy path to a SQLite database containing the FTS5 virtual table `chunks_fts` and a `chunks` table with at least `id`, `text`, `cluster`, `filename`, `section` columns. All three callers wrap the call in `if sources_db:`, so `None` is treated as "don't call this function."
- `question` should be a non-empty string; the callee tokenizes it via `re.findall(r'\w+', query)`. Passing `None` would raise `TypeError` inside `search_source_chunks` — **not caught here**.
- `top_k` should be a positive integer; it flows into `LIMIT ?` verbatim.

**Postconditions:**
- Returns a single markdown-formatted string, or `""`.
- Never raises for database-related failures (OperationalError/DatabaseError are swallowed). Other exceptions (e.g., `TypeError`, `ImportError`) are **not** swallowed.
- Inputs are not mutated.

**Invariants:**
- Every non-empty chunk in the output appears under one `### [N] filename` header (with optional ` > section` suffix).
- Chunks are separated by `\n\n---\n\n`.
- The returned string has no leading/trailing separator.

## Parameters

| Name | Meaning | Expected type / edge cases |
|---|---|---|
| `question` | Free-text query passed to FTS5 | `str`. Tokenization is the callee's job — stop-word filtering, length>1, OR-joined quoted terms. |
| `sources_db` | Path to the SQLite FTS index | `str` (truthy). If path doesn't exist, SQLite will **create an empty file** on connect, then the query fails with `OperationalError` (no such table) → returns `""`. Directory paths fail with `unable to open database file` → caught → `""`. |
| `top_k` | Row limit for `LIMIT ?` | `int`, default `10`. `0` → empty output. Negative → SQLite interprets as "no limit" (not enforced by the API). |

## Return Value

- **Non-empty string** — when at least one row matches and formatting runs:
  ```
  ### [1] {filename} > {section}

  {text}

  ---

  ### [2] {filename}

  {text}
  ```
  The ` > {section}` fragment is only emitted when `row["section"]` is truthy (so `None` and `""` are both omitted). The header is a single `### ` line; `> section` is not a separate heading — it lives inside the same H3 line.
- **`""` (empty string)** — when:
  - `sqlite3.OperationalError` or `sqlite3.DatabaseError` is raised by the callee, or
  - The callee returns an empty list (no matching words after stop-word filtering, or zero rows).

**Caller obligations:**
- Callers test truthiness (`if sources_context:`) to decide whether to append the section. Do **not** assume the output is non-empty.
- The output is a prompt fragment, not meant for UI display — chunk `[i]` indexes are meant as citations, but `FTS_RAG_PROMPT` instructs the LLM to cite by `[filename]`, so numbering is only surfaced for readability. This is a mild convention mismatch worth being aware of.

## Algorithm

1. **Call the search** — `rows = search_source_chunks(question, sources_db, top_k)` inside `try`. That function:
   - Tokenizes `question`, filters `api._STOP_WORDS` and 1-char tokens (falls back to any 1+ char token if all were filtered).
   - Builds `"word1" OR "word2" ...` as a safe FTS5 MATCH string.
   - Opens the DB, joins `chunks_fts` on `chunks.id = chunks_fts.rowid`, orders by `rank`, applies `LIMIT top_k`, returns `list[dict]`.
2. **Swallow DB errors** — `except (sqlite3.OperationalError, sqlite3.DatabaseError): return ""`. Note: `OperationalError` subclasses `DatabaseError`, so this clause is redundant-but-harmless.
3. **Empty-fast-path** — `if not rows: return ""`.
4. **Format** — `enumerate(rows, 1)` gives `i = 1..N`.
   - Build `header = f"[{i}] {row['filename']}"`.
   - If `row["section"]` is truthy, append `f" > {row['section']}"`.
   - Append body as `f"### {header}\n\n{row['text']}"` — note: `###` is prepended here, not in `header`.
5. **Join and return** — `"\n\n---\n\n".join(parts)`.

## Side Effects

- **No mutations** of arguments.
- **DB I/O**: opens a connection (in `search_source_chunks`) and reads from `chunks_fts`/`chunks`. Connection is closed in a `finally` block.
- **File creation side effect** (indirect, via `sqlite3.connect` in the callee): if `sources_db` points to a path that doesn't exist, SQLite creates a 0-byte file before the `SELECT` fails. The empty file is not cleaned up. So a "safe" error-swallowing call can still leave a stray file on disk.
- **No logging**; silent on error.

## Error Handling

| Exception | Where it originates | Behavior |
|---|---|---|
| `sqlite3.OperationalError` (missing table, corrupt DB, bad path) | `cur.execute(...)` or `sqlite3.connect` | Caught → `""` |
| `sqlite3.DatabaseError` (superclass of many sqlite errors) | Any sqlite call | Caught → `""` |
| `TypeError` (e.g., `sources_db=None`) | `sqlite3.connect` | **Propagates** — not caught |
| `ImportError` (e.g., `api._STOP_WORDS` missing) | `search_source_chunks` import | **Propagates** — not caught |
| `AttributeError` (e.g., row lacks `filename`) | Formatting loop | **Propagates** — because the loop is outside `try` |

The last point is important: the formatting loop is **not** inside the `try`, so any exception raised while reading `row["filename"]`, `row["section"]`, or `row["text"]` escapes to the caller. The callee guarantees these keys via its SELECT, so this is safe only because of an **unenforced contract** between the two functions.

## Usage Patterns

Typical call sites (all pass `sources_db` only when the user supplied `--full-sources`):

```python
if sources_db:
    sources_context = _search_source_chunks(question, sources_db)
    if sources_context:
        beliefs_context = beliefs_context + "\n\n## Source Documents\n\n" + sources_context
```

And in `_fts_rag_answer`:

```python
context = _search_source_chunks(question, sources_db)
if not context:
    return "No relevant documents found for this question."
```

**Caller obligation**: use an `if context:` (or `if sources_context:`) guard before interpolating into a prompt — an empty string would otherwise insert a dangling `"## Source Documents"` header with no content.

## Dependencies

- `search_source_chunks` (same module) — the search primitive.
- `sqlite3` (stdlib) — exception types only in this function.
- Indirectly, `reasons/api.py`'s `_STOP_WORDS` (imported inside `search_source_chunks`).
- No dependency on `reasons.llm` or `invoke_model` — this is presentation-only.

## Topics to Explore

- [function] `reasons/ask.py:search_source_chunks` — Underlying FTS5 search that builds the tokenized query, joins `chunks_fts` with `chunks`, and returns structured rows
- [function] `reasons/ask.py:_fts_rag_answer` — Consumer that renders this output into `FTS_RAG_PROMPT` and returns a "No relevant documents" fallback on empty context
- [file] `reasons/api.py` — Defines `_STOP_WORDS` used by the callee for query tokenization
- [general] `chunks-fts-schema` — The `chunks` table and FTS5 virtual table schema; determines what `filename`, `section`, and `rank` mean
- [general] `dual-answer-merge` — How the `dual=True` path combines TMS and FTS answers, of which this chunk context is one input

## Beliefs

- `search-source-chunks-swallows-db-errors` — Catches `sqlite3.OperationalError` and `sqlite3.DatabaseError` and returns `""` rather than propagating.
- `search-source-chunks-returns-markdown` — Returns a `### [i] filename[ > section]` markdown block per row, joined by `\n\n---\n\n`, or `""` when empty.
- `search-source-chunks-requires-fts5-schema` — Requires `chunks_fts` and `chunks` tables to exist in `sources_db`; the schema is not verified by this function.
- `search-source-chunks-has-file-creation-side-effect` — Because `sqlite3.connect` opens the target path, a nonexistent `sources_db` leaves a 0-byte file behind even on the error path.
- `search-source-chunks-enables-rag-context` — Its output is embedded under `## Source Documents` in `ask()` and as `{context}` in `_fts_rag_answer`, enabling source-grounded answers.
