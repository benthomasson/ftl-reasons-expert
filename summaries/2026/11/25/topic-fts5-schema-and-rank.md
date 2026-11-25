---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: How `chunks_fts` is indexed and how FTS5 `MATCH`, `rank`, and `rowid` behave.

## How `chunks_fts` is indexed

The observed `chunks_fts` definitions are FTS5 external-content tables:

```sql
CREATE VIRTUAL TABLE chunks_fts
USING fts5(text, content=chunks, content_rowid=id)
```

This appears in tests such as `tests/test_ask.py:365`, `tests/test_search_sources.py:23`, and `tests/test_report_belief.py:172`.

Key implications:

- The FTS index is over the `text` column only. Columns like `filename`, `section`, and `cluster` are not indexed by `chunks_fts`; they live in the ordinary `chunks` table and are retrieved by joining `chunks`.
- `content=chunks` makes `chunks_fts` an external-content FTS5 table backed by the `chunks` table.
- `content_rowid=id` maps the FTS5 rowid to `chunks.id`. So for this table, the FTS rowid is expected to correspond to `chunks.id`.
- No tokenizer is specified for `chunks_fts` in the observed definitions, so it uses SQLite FTS5's default tokenizer behavior. This is different from `nodes_fts` in `reasons/storage.py:71`, which explicitly uses `porter unicode61 tokenchars '-_'`.
- Because this is an external-content FTS table, inserts into `chunks` do not automatically populate the FTS index unless there are triggers or a rebuild/manual index-population step. The tests demonstrate both patterns:
  - `tests/test_ask.py:367` uses `INSERT INTO chunks_fts(chunks_fts) VALUES('rebuild')`.
  - `tests/test_search_sources.py:40-48` and `tests/test_report_belief.py:174` manually insert `(rowid, text)` into `chunks_fts`.

## How `MATCH` behaves

The query in `reasons/ask.py:240-243` uses:

```sql
FROM chunks_fts
JOIN chunks c ON c.id = chunks_fts.rowid
WHERE chunks_fts MATCH ?
ORDER BY chunks_fts.rank
```

`WHERE chunks_fts MATCH ?` is an FTS5 full-text search, not a normal substring comparison. It asks the FTS5 index for rows in `chunks_fts` whose indexed `text` matches the query string. SQLite then returns the matching FTS rowids.

That is why the join works:

```sql
JOIN chunks c ON c.id = chunks_fts.rowid
```

The FTS match returns `chunks_fts.rowid`, and `content_rowid=id` makes that correspond to `chunks.id`.

Also, because only `text` is indexed, a query like `"exported JSON"` matches the text content, not `filename` or `section`. The tests confirm this style of use in `tests/test_search_sources.py`, e.g. searching `"retraction"` and then reading `filename` and `section` from the joined `chunks` row.

The exact MATCH query construction is not shown in the observations. We can see the SQL placeholder `?`, but not how `search_source_chunks` builds the string, handles empty queries, stop words, quoting, or top-k limiting.

## How `rank` behaves

`ORDER BY chunks_fts.rank` in `reasons/ask.py:243` orders by FTS5's auxiliary `rank` column. In FTS5, `rank` is typically the BM25 relevance score, and lower scores are better. Since the query uses plain `ORDER BY chunks_fts.rank` with no `DESC`, SQLite defaults to ascending order, so the best matches are intended to come first.

The same pattern appears for `nodes_fts` in `reasons/api.py:3172`:

```sql
SELECT id FROM nodes_fts WHERE nodes_fts MATCH ? ORDER BY rank LIMIT ?
```

So the codebase uses the standard FTS5 relevance ordering convention.

`rank` is not a column stored in `chunks`; it is an FTS5 auxiliary column available on the virtual table for matched rows.

## How `rowid` behaves

`rowid` is the implicit FTS5 row identifier. For `chunks_fts`, the DDL option:

```sql
content_rowid=id
```

means the FTS rowid should match `chunks.id`.

That is why the production query in `reasons/ask.py:241` can do:

```sql
JOIN chunks c ON c.id = chunks_fts.rowid
```

and why the tests insert FTS rows with rowids matching the `chunks` rows:

```sql
INSERT INTO chunks_fts (rowid, text)
VALUES (1, 'Retraction cascades ...')
```

in `tests/test_search_sources.py:40-48`.

If the FTS rowid did not match `chunks.id`, the join would either drop the row or associate it with the wrong chunk metadata.

## What is missing from the observations

The observations are enough to explain the FTS5 mechanics, but not enough to fully explain production behavior:

- The production `CREATE VIRTUAL TABLE chunks_fts ...` statement was not found in `reasons/storage.py`; the observed `chunks_fts` DDL appears only in tests.
- The full `search_source_chunks` implementation in `reasons/ask.py` is not shown. We see the SQL fragment at lines 240-243, but not query preprocessing, stop-word handling, empty-query behavior, `top_k` LIMIT generation, or result mapping.
- The code that keeps `chunks_fts` synchronized with `chunks` in production is not visible. Tests use both `rebuild` and manual inserts, but production triggers/rebuild logic were not captured.
- The tokenizer for production `chunks_fts` is not observed. The test definitions do not specify one, but a production migration could differ.

## Topics to Explore

- [function] `reasons/ask.py:search_source_chunks` — The function that builds the MATCH query, applies `top_k`, executes the join, and maps rows into result dicts.
- [file] `reasons/ask.py` — Inspect the full SQL and surrounding logic around lines 240-243 to see how the FTS query string and LIMIT are constructed.
- [file] `reasons/storage.py` — Check whether production `chunks_fts` DDL, triggers, or rebuild logic exist outside the captured test definitions.
- [general] `chunks-fts-maintenance` — Search for `rebuild`, triggers on `chunks`, or inserts into `chunks_fts` to understand how the index stays synchronized.
- [general] `fts5-query-syntax` — Understand how SQLite FTS5 interprets MATCH strings, especially since the code passes a query parameter directly into `MATCH ?`.

## Beliefs

- `chunks-fts-is-external-content-table` — The observed `chunks_fts` definitions use FTS5 with `content=chunks`, so it is an external-content FTS table backed by `chunks`.
- `chunks-fts-indexes-only-text` — Only the `text` column is indexed by `chunks_fts`; `filename`, `section`, and `cluster` are read from `chunks` through the join.
- `chunks-fts-rowid-maps-to-chunks-id` — The query join `c.id = chunks_fts.rowid` depends on `content_rowid=id` mapping FTS rowids to `chunks.id`.
- `chunks-fts-rank-orders-best-first` — `ORDER BY chunks_fts.rank` uses FTS5's default ascending rank ordering, where lower BM25 scores indicate better matches.
- `chunks-fts-match-returns-fts-rowids` — `WHERE chunks_fts MATCH ?` filters using the FTS5 index on `text` and returns FTS rows whose rowids can be joined back to `chunks`.
