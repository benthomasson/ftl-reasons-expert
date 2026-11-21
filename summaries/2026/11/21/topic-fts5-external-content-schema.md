---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Understand why `chunks_fts` uses `content=chunks` and why the test helper manually synchronizes `chunks_fts`.

## Why `chunks_fts` uses `content=chunks`

`chunks_fts` is declared as an FTS5 **external-content** table:

```sql
CREATE VIRTUAL TABLE chunks_fts
USING fts5(text, content=chunks, content_rowid=id)
```

This appears in the tests at `tests/test_ask.py:365`, `tests/test_ask.py:620`, `tests/test_ask.py:732`, `tests/test_ask.py:785`, `tests/test_report_belief.py:172`, and `tests/test_search_sources.py:23`.

The `content=chunks` option tells FTS5 that the canonical text lives in the `chunks` table, not in `chunks_fts`. The FTS5 table stores the inverted index used for `MATCH` queries, but the actual content is read from `chunks`. `content_rowid=id` maps the FTS5 `rowid` to `chunks.id`.

That mapping is exactly what `reasons/ask.py:240-243` relies on:

```sql
FROM chunks_fts
JOIN chunks c ON c.id = chunks_fts.rowid
WHERE chunks_fts MATCH ?
ORDER BY chunks_fts.rank
```

So `chunks_fts` provides full-text search and ranking, while `chunks` remains the source of truth for chunk text and metadata. This avoids duplicating chunk text in the FTS table and keeps the join key explicit.

## Why the test setup manually synchronizes `chunks_fts`

FTS5 external-content tables are not automatically updated when the content table changes. If rows are inserted into `chunks`, `chunks_fts` does not automatically see them unless there are sync triggers or an explicit rebuild/index update.

The test setups create `chunks_fts` and then manually rebuild it:

- `tests/test_ask.py:365-367`
- `tests/test_ask.py:620-622`
- `tests/test_ask.py:732-734`
- `tests/test_ask.py:785-787`

using:

```sql
INSERT INTO chunks_fts(chunks_fts) VALUES('rebuild')
```

That `rebuild` command tells FTS5 to repopulate the index from the external content table, `chunks`.

Some tests instead seed the FTS index directly:

- `tests/test_report_belief.py:172-174`
- `tests/test_search_sources.py:23,40-48`

with `INSERT INTO chunks_fts(rowid, text) ...`. That is another manual way to populate or synchronize the FTS index for test data.

So the manual synchronization exists because the tests create and populate the tables themselves, and the observed code does not show trigger-based synchronization. Without a rebuild or direct FTS insert, `chunks_fts MATCH ?` would not reliably see the test data.

The observations do not show the production `chunks_fts` creation or any triggers on `chunks`, so the exact production synchronization mechanism is not visible here. The observations also do not show a standalone test helper function; they show repeated inline setup patterns that perform the synchronization.

## Topics to Explore

- [file] `reasons/ask.py` — See how `chunks_fts MATCH` and `chunks_fts.rank` are used in the search query.
- [file] `reasons/storage.py` — Check whether production schema creation includes `chunks_fts` triggers or rebuild logic.
- [file] `tests/test_ask.py` — Inspect the repeated schema setup and rebuild calls around the observed line numbers.
- [file] `tests/test_search_sources.py` — Compare direct `chunks_fts(rowid, text)` inserts with the `rebuild` approach.
- [general] `sqlite-fts5-external-content — Learn how FTS5 external-content tables and synchronization commands work.

## Beliefs

- `chunks-fts-uses-external-content` — `chunks_fts` is created with `content=chunks` and `content_rowid=id` in the observed test schemas.
- `ask-joins-fts-rowid-to-chunks-id` — `reasons/ask.py:240-243` joins `chunks_fts.rowid` to `chunks.id` while using `MATCH` and `rank`.
- `external-content-requires-explicit-sync` — An FTS5 external-content index is not automatically updated by inserts into its content table without triggers or explicit synchronization.
- `test-ask-setup-uses-rebuild` — `tests/test_ask.py` manually runs `INSERT INTO chunks_fts(chunks_fts) VALUES('rebuild')` after creating `chunks_fts`.
- `test-search-sources-directly-seeds-fts` — `tests/test_search_sources.py:40-48` manually inserts `(rowid, text)` rows into `chunks_fts`.
