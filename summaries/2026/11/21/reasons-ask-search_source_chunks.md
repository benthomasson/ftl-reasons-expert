---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: search_source_chunks in reasons/ask.py

## Purpose

`search_source_chunks` is the low-level FTS5 retrieval helper for source documents. It takes a natural-language query, converts it into an FTS5 `OR` query, searches the `chunks_fts` index, and returns the top matching source chunks as structured dictionaries.

It exists so higher-level code can retrieve source-document context without knowing the SQLite schema or FTS5 query syntax. The main caller is `_search_source_chunks`, which catches database errors and formats the returned rows into Markdown context for LLM prompts.

## Contract

**Preconditions**

- `query` should be a `str`. Passing `None` will raise from `re.findall`.
- `sources_db` should be a valid path to a SQLite database containing:
  - an FTS5 virtual table named `chunks_fts`
  - a table named `chunks` with columns `id`, `text`, `cluster`, `filename`, `section`
  - a relationship where `chunks_fts.rowid` corresponds to `chunks.id`
- `.api._STOP_WORDS` must exist and should contain lowercased stop-word strings.
- `top_k` is expected to be a non-negative or positive integer. It is not validated.

**Postconditions**

- Returns a `list[dict]`. Each dict contains the selected columns from `chunks`: `text`, `cluster`, `filename`, `section`.
- Returns `[]` if the query yields no usable words, or if the FTS query matches no rows.
- The SQLite connection is closed in a `finally` block after a connection has been created.

**Important invariant**

Stop words are removed only if at least one non-stop word of length greater than 1 remains. Otherwise, the function falls back to all raw words of length greater than 1, including stop words. So `"the of"` will search for `"the" OR "of"` if those are stop words, while `"the cat"` searches only for `"cat"`.

## Parameters

- `query`: Natural-language search string. Tokenized with `re.findall(r'\w+', query)`. This keeps Unicode word characters but drops punctuation, so `"C++"` becomes `["C"]`, which is then filtered out by the length check and may produce an empty result.
- `sources_db`: Path passed to `sqlite3.connect`. If the file does not exist, SQLite may create an empty database, and the subsequent query will usually fail because the expected tables do not exist.
- `top_k`: Default `10`. Passed as the SQL `LIMIT` parameter. `0` returns no rows. Negative values follow SQLite semantics and generally mean “no upper bound,” but the function does not document or enforce that.

## Return Value

Returns a list of dictionaries such as:

```python
[
    {
        "text": "...",
        "cluster": "...",
        "filename": "...",
        "section": "...",
    },
    ...
]
```

The docstring lists keys as `filename, section, text, cluster`, but the actual `SELECT` order is `text, cluster, filename, section`. Callers should rely on dictionary keys, not insertion order.

If no usable words remain after token filtering, the function returns `[]` before opening the database. If the FTS query matches nothing, it also returns `[]`.

Direct callers must handle SQLite exceptions. The wrapper `_search_source_chunks` catches SQLite database errors and turns them into an empty context string.

## Algorithm

1. Import `_STOP_WORDS` from `.api` at call time.
2. Tokenize `query` with `re.findall(r'\w+', query)`.
3. Keep words that:
   - are not in `_STOP_WORDS` after lowercasing
   - have length greater than 1
4. If that filtering produces no words, fall back to all raw words with length greater than 1.
5. If still no words, return `[]`.
6. Build an FTS5 query by quoting each word and joining with ` OR `:
   ```python
   '"word1" OR "word2" OR ...'
   ```
   Because `\w+` excludes FTS operators and quotes, the individual terms are safe to quote.
7. Connect to `sources_db` with `sqlite3.connect`.
8. Set `conn.row_factory = sqlite3.Row`.
9. Execute:
   ```sql
   SELECT c.text, c.cluster, c.filename, c.section
   FROM chunks_fts
   JOIN chunks c ON c.id = chunks_fts.rowid
   WHERE chunks_fts MATCH ?
   ORDER BY chunks_fts.rank
   LIMIT ?
   ```
10. Convert each `sqlite3.Row` to a plain `dict`.
11. Close the connection in `finally`.

The query is recall-oriented: because terms are joined with `OR`, a chunk can match if it contains any query term. FTS5 `rank` is used to order results, with lower rank values indicating better matches.

## Side Effects

- Opens a SQLite connection and reads from `sources_db`.
- Closes that connection in a `finally` block.
- Performs an import from `.api` on every call.
- Does not write to the database.
- Does not mutate `query`, `sources_db`, or `top_k`.

## Error Handling

`search_source_chunks` does not catch exceptions itself.

Possible exceptions include:

- `TypeError` if `query` is not a string compatible with `re.findall`.
- `ImportError` if `.api._STOP_WORDS` cannot be imported.
- `sqlite3.OperationalError` or other `sqlite3.DatabaseError` subclasses if:
  - the database cannot be opened
  - `chunks_fts` or `chunks` does not exist
  - FTS5 is unavailable
  - the schema does not match the query
  - `top_k` is invalid for SQLite
- SQLite may create a new empty database if `sources_db` points to a non-existent file, then fail when querying missing tables.

The caller `_search_source_chunks` wraps this function and catches `sqlite3.OperationalError` and `sqlite3.DatabaseError`, returning `""` on retrieval failure. That means higher-level `ask` flows degrade gracefully to “no source document context” when the source database is broken or missing. Direct callers do not get that behavior.

## Usage Patterns

Typical call chain:

```python
_search_source_chunks(question, sources_db, top_k=10)
    -> search_source_chunks(question, sources_db, top_k)
    -> format rows into Markdown source excerpts
```

`_search_source_chunks` formats each row as:

```markdown
### [1] filename > section

text
```

and joins multiple excerpts with `---`.

This function is used by `_fts_rag_answer` and by `ask` when `sources_db` is provided, including dual-summary and source-augmented answer modes. Callers that want graceful degradation should use `_search_source_chunks` rather than calling `search_source_chunks` directly.

## Dependencies

- Standard library: `re`, `sqlite3`
- Internal: `.api._STOP_WORDS`
- Database schema: `chunks_fts` FTS5 virtual table and `chunks` table
- SQLite FTS5 extension must be available

## Assumptions Not Enforced by Types

- `_STOP_WORDS` contains lowercased entries. The function lowercases candidate words but does not normalize stop words.
- `query` is a string. `None` is not handled.
- `top_k` is a sensible integer. Negative values have SQLite-specific behavior.
- `chunks_fts.rowid` equals `chunks.id`.
- `chunks_fts` exposes `rank` and supports `MATCH`.
- The FTS tokenizer treats quoted single-word phrases in the expected way.
- Duplicate query words are not deduplicated, so `"cat cat"` produces `"cat" OR "cat"`.
- Punctuation-only or single-character searches return `[]` without touching the database.

## Topics to Explore

- [function] `reasons/ask.py:_search_source_chunks` — Wraps this function, catches SQLite errors, and formats rows into Markdown source context.
- [file] `reasons/api.py` — Defines `_STOP_WORDS` and the belief-search API used alongside source-document retrieval.
- [file] `tests/test_search_sources.py` — Exercises stop-word filtering, result shape, and source-database failure behavior.
- [general] `fts5-schema-and-rank` — How `chunks_fts` is indexed and how FTS5 `MATCH`, `rank`, and `rowid` behave.
- [function] `reasons/ask.py:_fts_rag_answer` — Uses the formatted source chunks to synthesize an answer in FTS RAG mode.

## Beliefs

- `search-source-chunks-returns-structured-chunks` — The function returns a list of dictionaries with keys `text`, `cluster`, `filename`, and `section`, or `[]` when no usable tokens or matches exist.
- `search-source-chunks-uses-fts5-or-query` — It tokenizes with `\w+`, filters stop words and single-character words, then builds a quoted `OR` FTS5 query rather than an `AND` query.
- `search-source-chunks-stop-word-filtering-has-fallback` — Stop words are excluded only when at least one non-stop word remains; if all candidates are stop words, the fallback re-includes words of length greater than 1.
- `search-source-chunks-propagates-sqlite-errors` — It does not catch SQLite exceptions; `_search_source_chunks` is responsible for catching database errors and converting them to empty context.
- `search-source-chunks-assumes-chunks-fts-rowid-join` — The SQL assumes `chunks_fts.rowid` maps to `chunks.id` and that `chunks_fts` supports FTS5 `MATCH` and `rank`.
