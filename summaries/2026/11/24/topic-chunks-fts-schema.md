---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: The `chunks` table and FTS5 virtual table schema; determines what `filename`, `section`, and `rank` mean

## The `chunks` table and `chunks_fts` FTS5 virtual table

### Where it lives

`chunks` is **not** part of the main beliefs database (`reasons/storage.py` only defines `nodes`, `justifications`, `nodes_fts`, etc. — see `reasons/storage.py:71`). It lives in a **separate sources/RAG SQLite file** (e.g. `rag_fts.db`, referenced at `reasons/ask.py:389` and passed to search via `--sources-db`/`sources_db` in `reasons/cli.py:3611` and `reasons/api.py:3772`). The schema itself only appears verbatim in the test fixtures (`tests/test_ask.py:356`, `:599`, `:723`), so the canonical definition comes from test-created tables.

### The `chunks` table schema

Reconstructed from the test inserts:

- `tests/test_ask.py:729` — `INSERT INTO chunks VALUES (1, 'Summit is in June 2025', 'ev', 'events.md', 'Dates')`
- `tests/test_ask.py:362` — `INSERT INTO chunks VALUES (1, 'Important source doc content', 'docs', 'doc.md', NULL)`
- `tests/test_ask.py:608` — explicit column list `(id, text, cluster, filename, section)`

That gives a column order of:

```sql
CREATE TABLE chunks (
    id       INTEGER PRIMARY KEY,   -- rowid, matched via chunks_fts.content_rowid
    text     TEXT,                  -- the searchable chunk body
    cluster  TEXT,                  -- grouping label produced by the indexer
    filename TEXT,                  -- original source document name
    section  TEXT                   -- optional section heading within the document
);
```

The `id` is the **rowid** referenced by the FTS table (`content_rowid=id`), so each chunk row is uniquely addressable by FTS5.

### The `chunks_fts` FTS5 virtual table

```sql
CREATE VIRTUAL TABLE chunks_fts
USING fts5(text, content=chunks, content_rowid=id);
```

Observed at `tests/test_ask.py:365`, `:620`, `:732`, followed by:

```sql
INSERT INTO chunks_fts(chunks_fts) VALUES('rebuild');
```

Two important properties:

1. **External-content FTS5** (`content=chunks`): `chunks_fts` does *not* store its own copy of the text. It indexes `text` and reads the values back from the `chunks` table via `content_rowid=id`. This is why queries use a `JOIN`:
   ```sql
   SELECT c.text, c.cluster, c.filename, c.section
   FROM chunks_fts
   JOIN chunks c ON c.id = chunks_fts.rowid
   WHERE chunks_fts MATCH ?
   ORDER BY chunks_fts.rank
   ```
   (`reasons/ask.py:239–243`)

2. **Only `text` is indexed.** The `cluster`, `filename`, and `section` columns are *not* searched by `MATCH`; they are payload columns returned alongside hits (and threaded through the return dict per `reasons/ask.py:222`).

Note the contrast with the beliefs-side `nodes_fts` (`reasons/storage.py:71`, `:226`), which uses `tokenize="porter unicode61 tokenchars '-_'"`. The test fixture for `chunks_fts` does not specify a tokenizer, so it falls back to the FTS5 default (`unicode61`) — chunk search is not porter-stemmed by the schema as observed.

### What `filename`, `section`, and `rank` mean

**`filename`** — the name of the source document the chunk was extracted from (e.g. `doc.md`, `events.md`). It is the primary citation key:
- Used as the retrieval header: `header = f"[{i}] {row['filename']}"` (`reasons/ask.py:261`; echoed in `reasons/cli.py:1345`).
- Named as the citation format in the synthesis prompt: *"Cite sources using `[filename]` notation"* (`reasons/ask.py:273`, `:289`).

**`section`** — an optional heading/sub-unit within the document (e.g. `"Dates"`). It **can be NULL** (`tests/test_ask.py:362` inserts `NULL`). Display code appends it with a `>` separator only when present:
```python
header = f"[{i}] {row['filename']}"
if row["section"]:
    header += f" > {row['section']}"
```
(`reasons/ask.py:261–263`, mirrored in `reasons/cli.py:1346–1347` and `reasons/report_belief.py:103–105`). So the canonical citation string is `filename > section` when a section exists, otherwise just `filename`.

**`rank`** — the FTS5 hidden column that exposes the ranking function's score (BM25 by default in FTS5). It is *not* a stored column on `chunks`; `chunks_fts.rank` is provided by the virtual table. `ORDER BY chunks_fts.rank` (`reasons/ask.py:243`) sorts hits from most to least relevant (FTS5's default BM25 rank returns lower values for better matches). The same pattern is used for `nodes_fts` at `reasons/api.py:3172`.

### What the observations do *not* say

- The exact SQLite type/constraints on `chunks` (`INTEGER PRIMARY KEY` vs `TEXT`, indexes, etc.) — only inferred from test inserts.
- How `chunks` and `chunks_fts` are populated in production (the indexing/ingestion pipeline that writes `rag_fts.db` is not present in the observed snippets — `find_search_sources_symbol` returned 0, and the ingest side of the sources DB was not surfaced).
- The semantics of `cluster` beyond "a grouping label" — it is selected in `ask.py` but not used in citation formatting.
- Whether a `tokenize` option is expected on `chunks_fts` in production (tests omit it; storage's `nodes_fts` uses `porter unicode61`).

## Topics to Explore

- [file] `reasons/ask.py` — Contains `search_source_chunks`/FTS5 query construction and prompt-building surrounding `filename`/`section`/`rank`; read the full function around lines 220–300.
- [file] `tests/test_ask.py` — Authoritative fixture definitions for `chunks` and `chunks_fts`; the only in-repo DDL for these tables.
- [function] `reasons/ask.py:search_source_chunks` — The function that runs the `MATCH … ORDER BY chunks_fts.rank` query and returns the structured `{filename, section, text, cluster}` records.
- [general] `sources-db-ingestion` — How `rag_fts.db` gets populated (chunking, `cluster` assignment, `section` detection, `rebuild` of `chunks_fts`) — not covered by the current observations.
- [function] `reasons/api.py:search_source_chunks` (call site at line 3772) — How retrieved chunks are attached to beliefs as `source_chunks` and later rendered by `report_belief.py`.

## Beliefs

- `chunks-schema-is-id-text-cluster-filename-section` — The `chunks` table has the column order `(id, text, cluster, filename, section)` per test inserts at `tests/test_ask.py:362` and `:729`.
- `chunks-fts-uses-external-content` — `chunks_fts` is created with `fts5(text, content=chunks, content_rowid=id)`, making it an external-content FTS5 table that indexes only `text` and reads payload from `chunks`.
- `filename-and-section-compose-citation-header` — Retrieval headers concatenate `filename` and `section` with ` > ` only when `section` is truthy (`reasons/ask.py:261–263`).
- `section-is-nullable` — The `section` column accepts NULL (`tests/test_ask.py:362`), and display code guards it with `if row["section"]` before appending.
- `chunks-rank-maps-to-fts5-bm25` — `ORDER BY chunks_fts.rank` (`reasons/ask.py:243`) is FTS5's built-in rank column, which defaults to BM25-derived relevance scoring.
- `chunks-fts-lives-in-separate-sources-db` — `chunks`/`chunks_fts` belong to a separate sources/RAG SQLite file (`rag_fts.db`, `reasons/ask.py:389`), not the beliefs DB defined in `reasons/storage.py`.
