---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_search_sources.py

## Purpose

`tests/test_search_sources.py` owns regression coverage for the “search sources” feature. It verifies two surfaces:

1. `reasons.ask.search_source_chunks(...)` — the low-level function that searches an SQLite FTS5 chunks database.
2. `reasons.cli.cmd_search_sources(...)` — the CLI command that formats search results as text or JSON.

The tests are integration-style rather than mock-heavy: they create a real temporary SQLite database with the expected `chunks` and `chunks_fts` schema, run the production search/CLI code against it, and assert on returned values or captured stdout.

This file is test-only; production code does not import it.

## Key Components

### `_create_fts_db(path)`

A local helper that creates and returns a minimal searchable SQLite database at `path`.

It creates:

- `chunks(id INTEGER PRIMARY KEY, text TEXT, cluster TEXT, filename TEXT, section TEXT)`
- `chunks_fts`, an FTS5 virtual table:
  ```sql
  CREATE VIRTUAL TABLE chunks_fts USING fts5(
      text,
      content=chunks,
      content_rowid=id
  )
  ```

It inserts three rows:

| id | text | cluster | filename | section |
|---|---|---|---|---|
| 1 | `Retraction cascades propagate through the dependency graph` | `tms` | `architecture.md` | `Retraction` |
| 2 | `Nodes are justified by SL justifications with support lists` | `tms` | `architecture.md` | `Justifications` |
| 3 | `The belief network can be exported as JSON or Markdown` | `export` | `usage.md` | `""` |

It also manually inserts the same text into `chunks_fts` with matching rowids. This is important: because `chunks_fts` uses `content=chunks`, it is an external-content FTS table. The helper does not create triggers to keep the FTS index synchronized, so the test setup explicitly inserts both the content row and the FTS row.

Contract: returns the database path as a `str`.

### `TestSearchSourceChunks`

Tests the `search_source_chunks` function contract.

Notable cases:

- `test_basic_search`: searching `"retraction"` returns at least one result whose first item has `filename == "architecture.md"` and text containing `"Retraction"`.
- `test_returns_list_of_dicts`: result is a `list`, every element is a `dict`, and the first result has at least `text`, `filename`, and `section` keys.
- `test_no_matches`: a deliberately absent term returns exactly `[]`.
- `test_top_k_limits`: passing `top_k=1` limits result length to at most 1.
- `test_empty_query`: empty query returns `[]`.
- `test_stop_words_fallback_still_searches`: a query made only of stop words still returns a list and does not raise.
- `test_section_included`: section metadata is preserved for a matched row.
- `test_empty_section`: an empty section is preserved as `""`, not dropped or converted to `None`.
- `test_bad_db_raises`: pointing at a SQLite file without the expected FTS schema raises `sqlite3.OperationalError`.

Expected result shape for successful searches:

```python
[
    {
        "text": "...",
        "filename": "...",
        "section": "...",
        # possibly other keys
    },
    ...
]
```

### `TestSearchSourcesCli`

Tests `reasons.cli.cmd_search_sources(args)`.

The tests build a tiny dynamic `args` object using:

```python
args = type("Args", (), {
    "query": "retraction",
    "db": db,
    "top_k": 10,
    "format": "text",
})()
```

This mimics an `argparse.Namespace` without depending on real CLI parsing.

Cases:

- `test_text_output`: `format="text"` prints output containing `architecture.md` and `Retraction`.
- `test_json_output`: `format="json"` prints valid JSON; parsed output is a list and first item has `filename == "architecture.md"`.
- `test_no_results_message`: no matches in text mode prints a message containing `"No matching chunks"`.

## Patterns

- **Real SQLite integration tests**: The tests do not mock SQLite or monkeypatch search internals. They verify the actual FTS5 query path.
- **Local fixture factory**: `_create_fts_db` acts like a reusable fixture builder, but is called explicitly by each test with `tmp_path`.
- **Temporary filesystem isolation**: Every test gets a fresh database under `tmp_path`.
- **Class-based grouping**: Function tests and CLI tests are separated into `TestSearchSourceChunks` and `TestSearchSourcesCli`.
- **Ad-hoc CLI namespace**: `type("Args", (), {...})()` is used as a lightweight stand-in for `argparse.Namespace`.
- **Output capture with `capsys`**: CLI behavior is tested by capturing stdout rather than asserting on return values.
- **Manual FTS index synchronization in tests**: Because no FTS triggers exist in the helper, both `chunks` and `chunks_fts` are populated explicitly.

## Dependencies

### Imports

- `json` — parses CLI JSON output.
- `sqlite3` — creates/inspects the test database and asserts on `sqlite3.OperationalError`.
- `pytest` — provides `tmp_path`, `capsys`, and `pytest.raises`.
- `reasons.ask.search_source_chunks` — production function under test.
- `reasons.cli.cmd_search_sources` — imported locally inside each CLI test.

### Runtime/test dependencies

- SQLite must be compiled with FTS5 support.
- `tmp_path` must be writable.
- The production implementations of `search_source_chunks` and `cmd_search_sources` must exist and match the expected contracts.

### Imported by

Nothing else in the repository. This file is discovered and executed by pytest.

## Flow

1. A test calls `_create_fts_db(tmp_path / "chunks.db")`.
2. The helper creates the SQLite connection, creates `chunks` and `chunks_fts`, inserts three content rows and three FTS rows, commits, closes, and returns the path.
3. For function tests, `search_source_chunks(query, db, ...)` is called directly.
4. For CLI tests, an args-like object is constructed and passed to `cmd_search_sources(args)`.
5. Assertions inspect:
   - returned list/dict values,
   - result count,
   - metadata fields,
   - captured stdout,
   - parsed JSON output,
   - raised SQLite exceptions.

The data transformation under test is:

```text
query string
  -> search_source_chunks
  -> SQLite FTS5 lookup
  -> list of row dictionaries
  -> CLI formatter
  -> text or JSON output
```

## Invariants

- The search result contract includes at least `text`, `filename`, and `section` keys.
- A successful non-empty search returns a list, not a generator or custom object.
- No matches return `[]`, not `None`.
- Empty query returns `[]`.
- `top_k` limits the number of returned rows.
- Section values are preserved exactly, including empty strings.
- The CLI `format="json"` path emits valid JSON that parses to a list.
- The CLI `format="text"` path emits human-readable output containing filename and section information.
- A missing/unexpected FTS schema propagates as `sqlite3.OperationalError`; the test expects the production code not to swallow it.

## Error Handling

- `test_bad_db_raises` uses `pytest.raises(sqlite3.OperationalError)` and expects missing database schema errors to propagate from `search_source_chunks`.
- No matches are handled as a normal empty result by the search function.
- Stop-word-only queries are expected not to error; the test asserts they still return a list.
- The CLI handles zero search results gracefully by printing a `"No matching chunks"` message in text mode.
- This test file itself does not catch production errors beyond `pytest.raises`; it mostly asserts expected propagation or graceful no-result behavior.

## Topics to Explore

- [file] `reasons/ask.py` — Contains the production implementation of `search_source_chunks`, including query normalization and FTS5 SQL.
- [function] `reasons/ask.py:search_source_chunks` — Explore how stop words, empty queries, `top_k`, and result-row mapping are implemented.
- [function] `reasons/cli.py:cmd_search_sources` — Explore exact text/JSON formatting and no-results handling.
- [general] `fts5-external-content-schema` — Understand why `chunks_fts` uses `content=chunks` and why the test helper manually synchronizes `chunks_fts`.
- [general] `source-chunk-indexing-pipeline` — Find where production code creates/populates the `chunks` and `chunks_fts` tables, and whether triggers or rebuild commands are used.

## Beliefs

- `search-source-chunks-returns-list-of-dicts` — `search_source_chunks` returns a list of dictionaries containing at least `text`, `filename`, and `section` keys on successful matches.
- `empty-query-returns-empty-list` — `search_source_chunks("", db)` returns `[]`.
- `missing-fts-schema-raises-operational-error` — Passing a SQLite path without the expected `chunks_fts` table raises `sqlite3.OperationalError`.
- `cli-search-sources-supports-text-and-json` — `cmd_search_sources` emits either text or JSON depending on `args.format`.
- `test-helper-manually-populates-fts-index` — `_create_fts_db` inserts into both `chunks` and `chunks_fts` instead of relying on triggers.
