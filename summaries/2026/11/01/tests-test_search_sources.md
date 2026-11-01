---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_search_sources.py

# `tests/test_search_sources.py`

## 1. Purpose

This file is the integration test suite for the **full-text search over document chunks** feature in the `ftl-reasons` project. It owns two responsibilities:

1. **Contract testing** for `reasons.ask.search_source_chunks` — the library-level function that queries a SQLite FTS5 index of text chunks and returns ranked results.
2. **CLI output testing** for `reasons.cli.cmd_search_sources` — the argparse-wrapped command that formats and prints those results as either human-readable text or JSON.

In the broader project (a TMS/belief-network system), this search capability lets a user or agent retrieve relevant source-document passages by keyword, which feeds into reasoning, justification lookup, or wiki-building workflows.

## 2. Key Components

### `_create_fts_db(path) → str`
A standalone helper (not a pytest fixture) that builds a minimal but structurally faithful FTS5 database:

| Element | Detail |
|---|---|
| **`chunks` table** | `id`, `text`, `cluster`, `filename`, `section` — the "content table" |
| **`chunks_fts`** | `CREATE VIRTUAL TABLE … USING fts5(text, content=chunks, content_rowid=id)` — a *contentless* FTS5 index that reads back from `chunks` |
| **Seed data** | 3 rows spanning two clusters (`tms`, `export`), two filenames, and one empty `section` value |

It returns the DB path as a string. The function always commits and closes the connection before returning.

### `TestSearchSourceChunks`
Nine tests covering the library function's behavioral contract:

| Test | What it pins down |
|---|---|
| `test_basic_search` | A query hits at least one row; the first result's `filename` and `text` match expectations |
| `test_returns_list_of_dicts` | Return type is `list[dict]`; each dict contains keys `"text"`, `"filename"`, `"section"` |
| `test_no_matches` | Nonsense query → `[]` |
| `test_top_k_limits` | `top_k=1` caps results to ≤ 1 |
| `test_empty_query` | `""` → `[]` (guard against malformed FTS5 queries) |
| `test_stop_words_fallback_still_searches` | All-stopword query still returns a list (doesn't crash; may be empty) |
| `test_section_included` | Non-empty `section` column is surfaced in results |
| `test_empty_section` | Empty-string `section` is preserved (not converted to `None`) |
| `test_bad_db_raises` | Non-existent DB path → `sqlite3.OperationalError` (propagated, not swallowed) |

### `TestSearchSourcesCli`
Three tests for the CLI wrapper `cmd_search_sources`:

- **`test_text_output`** — calls the CLI with `format="text"`, asserts the stdout contains the filename and a snippet.
- **`test_json_output`** — same but `format="json"`; parses stdout as JSON and validates it's a list of dicts with the expected keys.
- **`test_no_results_message`** — a zero-hit query prints the literal string `"No matching chunks"`.

The CLI tests construct a fake argparse namespace via the idiom:
```python
args = type("Args", (), {
    "query": "retraction",
    "db": db,
    "top_k": 10,
    "format": "text",
})()
```
This is a lightweight stand-in for `argparse.Namespace` — it gives the CLI function a plain object with attribute access without importing `argparse`.

## 3. Patterns & Idioms

- **`tmp_path` fixture (not a custom one).** Every test creates its DB inside pytest's per-test `tmp_path` directory, guaranteeing isolation. No shared state, no teardown needed.
- **Helper-over-fixture for DB creation.** `_create_fts_db` is a plain function, not an `@pytest.fixture`. This keeps it visible and callable from any scope, and avoids the overhead of fixture parametrization when every test uses the same schema.
- **`capsys` for CLI I/O.** Instead of redirecting `sys.stdout` manually, the tests rely on pytest's `capsys` fixture to capture what `cmd_search_sources` prints.
- **Two test classes, two layers.** `TestSearchSourceChunks` exercises the *library* API; `TestSearchSourcesCli` exercises the *presentation* layer. This mirrors the `reasons/ask.py` → `reasons/cli.py` call boundary.
- **Inline imports in CLI tests.** `from reasons.cli import cmd_search_sources` is imported *inside* each test method rather than at module top level. This is a pragmatic choice: it keeps the module-level import list minimal and avoids a hard dependency on `reasons.cli` at collection time (useful if `cli.py` has heavy side-effects at import).
- **FTS5 content-table pattern.** The schema uses `content=chunks, content_rowid=id`, meaning the FTS5 index stores *only* the inverted index, and row data is read back from the `chunks` table. The tests seed both tables explicitly, mirroring what a production migration would do.

## 4. Dependencies

### This file depends on
| Import | Role |
|---|---|
| `reasons.ask.search_source_chunks` | The function under test (library layer) |
| `reasons.cli.cmd_search_sources` | The CLI handler under test (presentation layer) |
| `sqlite3` | Used in `_create_fts_db` to build the test database; also the expected exception type in `test_bad_db_raises` |
| `pytest` | `tmp_path`, `capsys`, `pytest.raises` |
| `json` | Parsing the JSON CLI output in `test_json_output` |

### What depends on this file
Nothing imports this file. It is a leaf test module. It is discovered and executed by `pytest` (via `pyproject.toml` configuration or `conftest.py`).

## 5. Flow

```
test setup
  │
  ├─ tmp_path / "chunks.db"  ← pytest gives a unique temp dir
  │
  ├─ _create_fts_db(path)
  │     ├─ sqlite3.connect → CREATE TABLE chunks
  │     ├─ CREATE VIRTUAL TABLE chunks_fts (FTS5, content-backed)
  │     ├─ INSERT 3 rows into chunks
  │     ├─ INSERT 3 rows into chunks_fts (explicit, to populate the index)
  │     ├─ conn.commit(); conn.close()
  │     └─ return path string
  │
  ▼
test execution
  │
  ├─ [library tests]
  │     search_source_chunks(query, db_path, top_k?)
  │       → opens its own sqlite3 connection
  │       → runs FTS5 MATCH query
  │       → JOINs back to chunks for filename, section, cluster
  │       → returns list[dict]
  │     assertions on shape, content, edge cases
  │
  └─ [CLI tests]
        cmd_search_sources(args_namespace)
          → calls search_source_chunks internally
          → formats results (text table or json.dumps)
          → prints to stdout
        capsys captures stdout → assertions on printed output
```

The data transformation is: **raw FTS5 rowid → JOIN with content table → list of dicts → (optional) CLI formatting → stdout**.

## 6. Invariants Enforced by These Tests

1. **Return shape**: `search_source_chunks` always returns `list[dict]`, never `None`, never a bare string.
2. **Required keys**: Every result dict must contain at minimum `"text"`, `"filename"`, `"section"`.
3. **Empty section is `""`**, not `None` — `test_empty_section` pins this.
4. **Empty/stopword queries degrade gracefully** to `[]` rather than raising an FTS5 syntax error.
5. **`top_k` is an upper bound**, not an exact count.
6. **Missing DB is a hard error** (`sqlite3.OperationalError`), not a silent empty result. The caller is expected to handle or surface this.
7. **CLI with zero results prints a human-readable message** (`"No matching chunks"`) rather than an empty line or `[]`.

## 7. Error Handling

- **`test_bad_db_raises`** is the *only* test that exercises the error path. It uses `pytest.raises(sqlite3.OperationalError)` to confirm that passing a non-existent DB path propagates the standard library exception. This tells the developer:
  - `search_source_chunks` does **not** catch or wrap the `sqlite3` error.
  - There is no custom exception type (e.g., `ReasonsError`) in this code path.
  - The CLI layer (`cmd_search_sources`) is *not* tested for the bad-DB case, so it's unclear whether the CLI catches this or lets it bubble to the top-level CLI error handler.

- **Stop-word / empty query**: The tests assert a *list* is returned (possibly empty) but do **not** assert it's specifically `[]`. The `test_stop_words_fallback_still_searches` test only checks `isinstance(results, list)`, implying the implementation may strip stop words and issue a broader query, returning some results. This is intentionally loose.

- **No `finally` or context-manager cleanup** in `_create_fts_db` beyond `conn.close()`. Since `tmp_path` is cleaned up by pytest after each test, leaking the file handle is a non-issue in practice, but it would be a concern if the helper were reused outside pytest.

## Topics to Explore

- [function] `reasons/ask.py:search_source_chunks` — The function under test; understanding its FTS5 query construction, stop-word filtering, and `top_k` slicing logic is essential before modifying these tests.
- [function] `reasons/cli.py:cmd_search_sources` — The CLI handler that formats results; reveals the text-vs-JSON formatting contract and how it handles zero-result and error cases.
- [file] `tests/conftest.py` — May define shared fixtures (e.g., a pre-built belief network or session-scoped DB) that other test files rely on but this file deliberately avoids, keeping its tests hermetic.
- [general] `fts5-content-table-migration` — How the production `chunks` + `chunks_fts` schema is created/migrated (likely in `reasons/storage.py` or `reasons/pg.py`), and whether the test helper `_create_fts_db` mirrors that exactly or is a simplified variant.
- [file] `reasons/cluster.py` — The `cluster` column in the `chunks` table is seeded with values like `'tms'` and `'export'`; understanding how clustering populates this field helps interpret search-result relevance ranking.

## Beliefs

- `search-source-chunks-returns-list-of-dicts` — `search_source_chunks` always returns a `list[dict]` (possibly empty) and never raises for empty or stop-word-only queries; only a missing/unreadable DB file causes an exception.
- `fts5-index-is-content-backed` — The `chunks_fts` virtual table uses `content=chunks, content_rowid=id`, meaning it stores no row data itself and depends on the `chunks` table existing for all non-`text` columns (`filename`, `section`, `cluster`).
- `test-helper-mirrors-production-schema` — `_create_fts_db` in the test file creates the same table/FTS5-virtual-table structure that production uses; if the production schema gains a column, this helper must be updated in lockstep or tests will silently pass with missing data.
- `cli-zero-results-produces-message-not-empty` — `cmd_search_sources` prints the string `"No matching chunks"` to stdout when the result list is empty, rather than printing nothing or an empty JSON array, making the no-hit case visible in terminal output.
- `test-file-is-hermetic` — This test file creates its own SQLite database per test via `tmp_path` and imports no shared fixtures from `conftest.py`, meaning it can be run in isolation without other test infrastructure.
