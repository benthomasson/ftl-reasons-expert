---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Understand the `nodes_fts` DROP/CREATE/INSERT approach as the repo's canonical way of keeping a contentless FTS table in sync

## How `nodes_fts` is kept in sync

First, a terminology correction: the observed `nodes_fts` table is **not** a SQLite FTS5 “contentless” table. It is created as a regular FTS5 virtual table with two stored/indexed columns:

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS nodes_fts
USING fts5(id, text, tokenize="porter unicode61 tokenchars '-_'");
```

That is in `reasons/storage.py:71`. There is no `content=''` and no `content=nodes` external-content clause. So if the repo calls this “contentless,” it likely means “derived index, not the source of truth,” not SQLite’s technical contentless-FTS5 mode.

### The canonical sync path: full rebuild in `Storage.save`

The sync strategy lives in `reasons/storage.py:Storage.save`. This is a full-network snapshot writer: it clears and rewrites the tables for small networks. Inside the `with self.conn:` transaction it does:

1. Snapshot externally added tags/sources.
2. Clear `node_tags`, `node_sources`, `justifications`.
3. Drop and recreate the FTS table:

```python
self.conn.execute("DROP TABLE IF EXISTS nodes_fts")
self.conn.execute(
    'CREATE VIRTUAL TABLE nodes_fts USING fts5(id, text, tokenize="porter unicode61 tokenchars \'-_\'")'
)
```

That is `reasons/storage.py:225-226`.

4. Clear and rewrite `nodes`, `nogoods`, `repos`, `propagation_log`, `network_meta`.
5. For each node in `network.nodes.values()`, insert into `nodes` and then insert the search row:

```python
self.conn.execute(
    "INSERT INTO nodes_fts (id, text) VALUES (?, ?)",
    (node.id, node.text),
)
```

That is `reasons/storage.py:258`.

So the “canonical” pattern is:

- Do **not** use triggers.
- Do **not** use incremental FTS updates.
- Do **not** use FTS5’s `rebuild` command.
- On every `save`, rebuild the FTS table from scratch from the in-memory `Network`.
- Do it inside the same SQLite transaction as the rest of the snapshot write, so a failed save can roll back the whole operation.

This is consistent with `Storage.save` already being a full rewrite. Since `nodes` is deleted and repopulated anyway, dropping and recreating `nodes_fts` is a simple way to guarantee the index exactly matches `network.nodes`. It also means the tokenizer definition is reset on every save, which matters because `_init_schema` only uses `CREATE VIRTUAL TABLE IF NOT EXISTS` at `reasons/storage.py:71`; an existing database would otherwise keep an old tokenizer configuration.

### How the table is queried

The search path uses the FTS table directly:

```sql
SELECT id FROM nodes_fts WHERE nodes_fts MATCH ? ORDER BY rank LIMIT ?
```

That is `reasons/api.py:3172`. The FTS table stores `id` and `text`; the query returns matching node IDs and orders by FTS5’s hidden `rank` column. The authoritative content is still `nodes.text`; `nodes_fts` is a derived search index.

One detail: `Storage.load` does **not** rebuild `nodes_fts`. It loads `nodes`, justifications, nogoods, metadata, repos, and log, then rebuilds the in-memory dependent index via `network._rebuild_dependents()`. FTS synchronization happens on `save`, not `load`. If the database’s FTS table is stale, loading the network alone does not fix it.

### Other backends and tests use different patterns

PostgreSQL does not use `nodes_fts`. It uses a GIN index on `to_tsvector('english', text)`:

```sql
CREATE INDEX IF NOT EXISTS idx_rms_nodes_fts
ON rms_nodes USING gin(to_tsvector('english', text));
```

That is `reasons/pg.py:84`, and `PgApi.search` uses `to_tsvector`/`plainto_tsquery` directly.

The tests that mention `chunks_fts` use a different pattern: external-content FTS5 with `content=chunks, content_rowid=id` and an FTS5 `rebuild` command, for example in `tests/test_ask.py:365-367` and `tests/test_search_sources.py:23,40`. Those are test fixtures for chunk search, not the canonical `nodes_fts` path.

The observed `tests/test_storage.py` covers save/load round trips, but it does not directly assert that `nodes_fts` contains the expected search rows after `Storage.save`. So the DROP/CREATE/INSERT behavior itself is not directly tested in the observations provided.

### What is missing from the observations

To fully verify the search behavior, we would need more context around `reasons/api.py:3172`: how the MATCH query string is built, whether searches are limited to the `text` column or the whole FTS table, how results are joined back to `nodes`, and whether `id` being indexed as a normal FTS column is intentional. We also do not see any direct test asserting `nodes_fts` MATCH results after `Storage.save`.

## Topics to Explore

- [file] `reasons/storage.py` — Inspect `_init_schema` and `save` together to see when `nodes_fts` is created versus fully rebuilt.
- [function] `reasons/storage.py:Storage.save` — Confirm the DROP/CREATE/INSERT sequence and how it fits inside the snapshot transaction.
- [file] `reasons/api.py` — Trace how the `nodes_fts MATCH` query at line 3172 builds its query string and maps results back to nodes.
- [function] `reasons/pg.py:PgApi.search` — Compare SQLite FTS5 search with PostgreSQL `to_tsvector`/`plainto_tsquery` search.
- [general] `nodes-fts-test-coverage` — Check whether any tests directly assert `nodes_fts` contents or MATCH behavior after `Storage.save`, since the observed tests only cover round-trip state and separate `chunks_fts` fixtures.

## Beliefs

- `nodes-fts-is-regular-fts5-table` — `nodes_fts` is created as `fts5(id, text, ...)` with no `content=` option, so it is a regular FTS5 table that stores its own `id` and `text` rather than a technical contentless or external-content table.
- `storage-save-rebuilds-nodes-fts` — `Storage.save` drops and recreates `nodes_fts` at `reasons/storage.py:225-226` and inserts each node’s `(id, text)` at `reasons/storage.py:258`.
- `nodes-fts-is-derived-from-in-memory-network` — The FTS rows are rebuilt from `network.nodes.values()` during `Storage.save`; no triggers or incremental FTS update path are used.
- `nodes-fts-query-uses-match-and-rank` — `reasons/api.py:3172` searches `nodes_fts` with `MATCH` and orders results by FTS5 `rank`.
- `pg-search-does-not-use-nodes-fts` — PostgreSQL search uses a GIN index on `to_tsvector('english', text)` at `reasons/pg.py:84`, not an FTS5 `nodes_fts` table.
