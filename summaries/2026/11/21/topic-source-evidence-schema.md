---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: How `chunks` and `chunks_fts` are populated and queried to attach source evidence to premises.

## How `chunks` and `chunks_fts` are populated and queried

### 1. What the two tables are

`chunks` is a normal SQLite table holding source-document chunks. In the test fixtures it has this shape:

```sql
CREATE TABLE chunks (
    id INTEGER PRIMARY KEY,
    text TEXT,
    cluster TEXT,
    filename TEXT,
    section TEXT
)
```

See `tests/test_search_sources.py:23` for the FTS setup and the inserts around `tests/test_search_sources.py:25-48`. Additional examples appear in `tests/test_ask.py:362`, `tests/test_ask.py:608-616`, `tests/test_ask.py:729`, and `tests/test_ask.py:782`.

`chunks_fts` is an FTS5 external-content index over `chunks.text`:

```sql
CREATE VIRTUAL TABLE chunks_fts
USING fts5(text, content=chunks, content_rowid=id)
```

Observed in `tests/test_search_sources.py:23`, `tests/test_ask.py:365`, `tests/test_ask.py:620`, `tests/test_ask.py:732`, `tests/test_ask.py:785`, and `tests/test_report_belief.py:172`.

That means:

- `chunks.text` is the canonical stored text.
- `chunks_fts` indexes that text for full-text search.
- The FTS `rowid` maps to `chunks.id` via `content_rowid=id`.

### 2. How they are populated

The observed population path is only in tests. The pattern is usually:

1. Insert rows into `chunks`.
2. Rebuild the external-content FTS index:

```sql
INSERT INTO chunks_fts(chunks_fts) VALUES('rebuild')
```

Examples:

- `tests/test_ask.py:362-367`
- `tests/test_ask.py:608-622`
- `tests/test_ask.py:729-734`
- `tests/test_ask.py:782-787`

Some tests insert into `chunks_fts` directly:

```sql
INSERT INTO chunks_fts(rowid, text) VALUES (...)
```

Examples:

- `tests/test_search_sources.py:40-48`
- `tests/test_report_belief.py:174`

Because no triggers are observed, the external-content FTS index must be refreshed either by a `rebuild` command or by explicit inserts into `chunks_fts`. The observations do not show a production ingestion path that creates `chunks` rows or refreshes `chunks_fts`. In particular:

- `find_chunks_inserts` only found `INSERT INTO chunks` in tests.
- `reasons/storage.py` defines `nodes`, `nodes_fts`, `node_sources`, etc., but not `chunks` or `chunks_fts`.

So the production code that populates the source-chunk database is missing from these observations.

### 3. How they are queried

The query is in `reasons/ask.py:240-243`:

```sql
FROM chunks_fts
JOIN chunks c ON c.id = chunks_fts.rowid
WHERE chunks_fts MATCH ?
ORDER BY chunks_fts.rank
```

This does three important things:

1. Searches the FTS index with `MATCH ?`.
2. Joins each FTS hit back to its `chunks` row using `chunks_fts.rowid = chunks.id`.
3. Orders results by FTS rank, so the best matches come first.

The tests import `search_source_chunks` from `reasons.ask` (`tests/test_search_sources.py:7`) and confirm the expected contract:

- Results are a list of dictionaries.
- Each result contains at least `text`, `filename`, and `section`.
- `top_k` limits the number of results.
- Empty queries return `[]`.
- Missing/bad DB paths raise `sqlite3.OperationalError`.
- Stop-word-only queries still return a list.

See `tests/test_search_sources.py:55-110`. The observed SQL does not show the `top_k` limit directly, so `top_k` is presumably applied inside `search_source_chunks` or by limiting/slicing the query results. The full function body was not included in the observations.

### 4. How this attaches source evidence to premises

The observed chunk search produces evidence snippets with provenance:

- `text` — the matched source excerpt
- `filename` — source document
- `section` — source section/heading

That is the “evidence” side.

The “premise” side is represented by network nodes. Storage defines `node_sources` in `reasons/storage.py`:

```sql
CREATE TABLE IF NOT EXISTS node_sources (
    node_id TEXT NOT NULL REFERENCES nodes(id),
    source_type TEXT NOT NULL DEFAULT '',
    source_ref TEXT NOT NULL DEFAULT '',
    source_url TEXT DEFAULT '',
    source_hash TEXT DEFAULT '',
    pinned_sha TEXT DEFAULT '',
    pinned_lines TEXT DEFAULT '',
    ...
)
```

This is the observed table that attaches source metadata to a node/premise ID. `pin_sources` also attaches git-aware provenance to nodes:

- `reasons/check_stale.py:311`
- `reasons/api.py:2784`

The tests in `tests/test_pin_sources.py` show that `pin_sources`:

- Operates on `Network` nodes with a `source` path.
- Skips OUT nodes, nodes without sources, and nonexistent sources.
- Backfills `source_hash`.
- Stores `pinned_sha` and `verified_at` in node metadata.
- Can rewrite GitHub source URLs to a pinned SHA.

However, the observations do **not** show the direct bridge between `chunks` search results and `node_sources` or node metadata. There is no observed schema column such as `chunks.node_id`, and no observed function that takes a `search_source_chunks` result and writes it into `node_sources`. So the exact “attach chunk search result to premise” step is not visible. What is visible is:

- `search_source_chunks` finds source evidence in `chunks`/`chunks_fts`.
- `node_sources` and `pin_sources` attach source provenance to nodes/premises.
- The code that connects those two flows is missing from the provided observations.

## Topics to Explore

- [file] `reasons/ask.py` — Inspect the full `search_source_chunks` function to confirm how `top_k`, stop-word fallback, output mapping, and error handling are implemented.
- [file] `reasons/check_stale.py` — Review `pin_sources` and related helpers to understand how git SHA, file hashes, and line ranges are attached to premises.
- [general] `chunk-ingestion-pipeline` — Locate production code that populates `chunks` and refreshes `chunks_fts`; the observations only show test fixtures.
- [file] `reasons/storage.py` — Study `node_sources` schema and `save`/`load` behavior to see how source evidence is persisted against node IDs.
- [general] `chunks-to-premises-link` — Find where search results are converted into attached evidence, since no direct `chunks.id` to `node_id` link is observed.

## Beliefs

- `chunks-fts-is-external-content-fts5` — `chunks_fts` is an FTS5 external-content table over `chunks.text`, keyed by `chunks.id` via `content_rowid=id`.
- `chunks-fts-requires-rebuild-or-direct-insert` — The observed tests populate `chunks_fts` either with `INSERT INTO chunks_fts(chunks_fts) VALUES('rebuild')` or by direct `chunks_fts(rowid, text)` inserts; no triggers are observed.
- `search-source-chunks-joins-fts-to-chunks` — The query in `reasons/ask.py:240-243` joins `chunks_fts` to `chunks` on `c.id = chunks_fts.rowid` and filters with `chunks_fts MATCH ?`.
- `chunks-query-returns-filename-and-section` — `search_source_chunks` results are tested to contain at least `text`, `filename`, and `section`.
- `node-sources-maps-evidence-to-premises` — `node_sources` is the observed storage table that associates source metadata with a `node_id`.
- `chunks-to-premises-link-is-not-observed` — No observed schema or code directly links `chunks.id` to a node/premise ID or writes chunk search results into `node_sources`.
