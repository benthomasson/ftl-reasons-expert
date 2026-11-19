---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_topics.py

# Explanation of `tests/test_topics.py`

## Purpose

This file is the test suite for the `topics` feature. It verifies both layers:

1. **API layer** — `reasons.api.topics(...)` discovers topic-like words from node IDs, filters noise, counts frequency, sorts, limits, and reports total node count.
2. **CLI layer** — `reasons topics ...` exposes that API through the command line, including human-readable output, JSON output, `--limit`, `--help`, and empty-database behavior.

It is an integration-style test file: it uses real temporary SQLite DB paths via pytest’s `tmp_path`, seeds nodes through `api.add_node`, and exercises the actual CLI entry point rather than mocking the internals.

---

## Key Components

### `run_cli(*args, db_path=None)`

Module-level helper for invoking the CLI in-process.

Contract:

```python
stdout, stderr, code = run_cli("topics", "--json", db_path=db)
```

It:

- Builds `sys.argv` as:
  - `["reasons"]`
  - optionally appends `["--db", db_path]`
  - appends the provided CLI args
- Redirects `sys.stdout` and `sys.stderr` into `StringIO`
- Patches `sys.argv`, `sys.stdout`, and `sys.stderr`
- Calls `reasons.cli.main`
- Catches `SystemExit`
- Returns `(stdout_text, stderr_text, exit_code)`

Important detail: when `db_path` is supplied, `--db` is inserted **before** the subcommand. For example:

```python
run_cli("topics", "--limit", "3", db_path=db)
```

becomes approximately:

```python
["reasons", "--db", db, "topics", "--limit", "3"]
```

So these tests encode the expectation that `--db` is accepted as a global CLI option before `topics`.

It does **not** catch arbitrary exceptions; only `SystemExit` is handled. If `main()` raises anything else, the test fails with a traceback.

---

### `TestTopicsApi`

Tests the API directly via `reasons.api.topics`.

Tests cover:

- `test_empty_db` — empty DB returns `{"topics": [], "total_nodes": 0}`.
- `test_extracts_words_from_ids` — topics are extracted from node IDs such as `"belief-about-caching"` and `"caching-strategy-redis"`; `"caching"` appears with `count == 2`.
- `test_filters_stop_words` — words like `"the"` and `"use"` are omitted.
- `test_filters_short_words` — two-character tokens like `"ab"` and `"cd"` are omitted.
- `test_splits_on_delimiters` — `"module.class:method_name"` yields `"module"`, `"class"`, `"method"`, and `"name"`.
- `test_limit` — `api.topics(limit=5, ...)` returns at most 5 topic entries.
- `test_sorted_by_frequency` — `"alpha"` with count 3 is first.
- `test_total_nodes` — `total_nodes` equals the number of inserted nodes.

The API result shape exercised by the tests is:

```python
{
    "topics": [
        {"topic": "caching", "count": 2},
        ...
    ],
    "total_nodes": 2,
}
```

The tests directly access `t["topic"]` and `t["count"]`, so those keys are part of the expected API contract.

---

### `TestCmdTopics`

Tests the CLI command through `run_cli`.

Tests cover:

- `test_help` — `topics --help` exits 0 and prints help containing `"topics"`.
- `test_basic_output` — normal human output contains topic names such as `"belief"` and the phrase `"topics from"`.
- `test_json_output` — `--json` emits valid JSON containing `"topics"` and `"total_nodes"`.
- `test_limit_flag` — `--limit 3` is reflected in human output as `"3 topics from"`.
- `test_empty_db` — empty DB prints `"No topics found"` and exits 0.

The CLI tests mostly use substring assertions rather than exact output matching, so the text format can evolve as long as those substrings remain.

---

### Imports

- `json` — parses CLI JSON output.
- `sys` — patched for CLI argv/stdout/stderr.
- `StringIO` — captures CLI output.
- `patch` — context manager for patching `sys`.
- `pytest` — imported, but not directly referenced; `tmp_path` is injected by pytest.
- `from reasons import api` — API under test and helper used to seed nodes.
- `from reasons.cli import main` — imported inside `run_cli`, not at module import time.

The repository also contains a `reasons_lib` package tree, but these tests import `reasons`, not `reasons_lib`.

---

## Patterns

- **Two-layer testing**: API tests validate data extraction/aggregation; CLI tests validate argument parsing and output formatting.
- **Arrange-Act-Assert**: each test creates a temp DB, inserts nodes, calls the target function, then asserts on the result.
- **Real temporary storage**: `tmp_path` provides a unique DB path per test, avoiding shared state and manual cleanup.
- **In-process CLI harness**: instead of spawning subprocesses, `run_cli` patches `sys.argv` and captures `stdout`/`stderr`.
- **API as setup helper**: CLI tests seed data through `api.add_node`, so the CLI tests depend on the storage/API behavior as well.
- **JSON contract testing**: `--json` output is parsed with `json.loads`, ensuring stdout is machine-parseable.
- **Substring assertions for human output**: formatting is tested loosely to avoid brittle exact-output checks.

---

## Dependencies

Depends on:

- `reasons.api.add_node` — used to seed test nodes.
- `reasons.api.topics` — API under test.
- `reasons.cli.main` — CLI entry point under test.
- pytest’s `tmp_path` fixture — provides isolated temporary directories/DB paths.
- Python standard library: `json`, `sys`, `io.StringIO`, `unittest.mock.patch`.

Depended on by:

- Pytest test discovery only. No application code imports this test module.

---

## Flow

For API tests:

1. Pytest injects `tmp_path`.
2. The test constructs a DB path: `str(tmp_path / "test.db")`.
3. The test seeds one or more nodes with `api.add_node(node_id, text, db_path=db)`.
4. The test calls `api.topics(db_path=db)` or `api.topics(limit=..., db_path=db)`.
5. Assertions inspect `result["topics"]`, topic names/counts, and `result["total_nodes"]`.

For CLI tests:

1. The test creates a temp DB path and seeds nodes.
2. `run_cli(...)` builds a fake `sys.argv`.
3. `run_cli` patches `sys.argv`, `sys.stdout`, and `sys.stderr`.
4. `run_cli` calls `reasons.cli.main()`.
5. `main()` parses the command, invokes the `topics` command, and prints either human-readable or JSON output.
6. If `main()` raises `SystemExit` — for example from `--help` — `run_cli` captures the exit code.
7. The test asserts on captured stdout, parsed JSON, and exit code.

---

## Invariants

These tests enforce or assume the following:

- `api.topics` returns a dictionary containing at least:
  - `"topics"` — a list of topic dictionaries
  - `"total_nodes"` — an integer
- Each topic dictionary contains:
  - `"topic"` — string
  - `"count"` — integer
- An empty DB yields:
  - `result["topics"] == []`
  - `result["total_nodes"] == 0`
- Topics are derived from **node IDs**, not from belief body text.
- Node IDs are tokenized on delimiters including `-`, `.`, `:`, and `_`.
- Tokens `"the"` and `"use"` are filtered as stop words.
- Two-character tokens such as `"ab"` and `"cd"` are filtered out.
- Topic frequency is counted across node IDs; for example `"caching"` appears in two IDs and gets `count == 2`.
- Topics are sorted by frequency descending; `"alpha"` with count 3 appears first.
- `limit` caps the number of returned topics.
- `total_nodes` counts inserted nodes.
- CLI `--json` prints valid JSON to stdout with `"topics"` and `"total_nodes"`.
- CLI human output includes phrases like `"topics from"` and `"3 topics from"`.
- CLI empty output includes `"No topics found"`.
- `reasons topics --help` exits with code 0 and prints help containing `"topics"`.
- `run_cli` treats normal `main()` return and `SystemExit(None)` as exit code `0`.
- `--db` must work as a global option before the subcommand, because `run_cli` places it before `topics`.

---

## Error Handling

- `run_cli` catches only `SystemExit`. This is necessary because argparse and CLI commands commonly raise `SystemExit` for `--help` or parse failures.
- `SystemExit` with `code is None` is normalized to `0`.
- `run_cli` returns `stderr` but no test currently asserts on it. Warnings or error text written to stderr will not fail a test unless they also affect exit code or stdout assertions.
- Non-`SystemExit` exceptions propagate and fail the test.
- There are no negative-path tests for invalid `--limit` values, malformed DB paths, or CLI parsing errors.
- API exceptions from `api.topics` or `api.add_node` are not caught by these tests.

---

## Topics to Explore

- [file] `reasons/api.py` — Contains `add_node` and `topics`; inspect tokenization, stop-word filtering, delimiter handling, sorting, and `total_nodes` behavior.
- [file] `reasons/cli.py` — Contains `main` and the `topics` command; inspect global `--db` parsing, `--json`, `--limit`, and human-readable output formatting.
- [function] `reasons/api.py:topics` — Verify the exact topic extraction algorithm and output schema asserted by this test file.
- [general] `topics-cli-output-format` — Compare the expected human strings `"topics from"`, `"3 topics from"`, and `"No topics found"` against the implementation.
- [general] `reasons-vs-reasons_lib` — The repository contains both `reasons` and `reasons_lib`; determine which package is canonical and why these tests import `reasons`.

## Beliefs

- `api-topics-returns-sorted-topics-and-total-nodes` — `reasons.api.topics` returns a dictionary with `topics` sorted by frequency descending and a `total_nodes` count; an empty DB yields empty topics and zero nodes.
- `topics-are-derived-from-node-ids` — The tests seed only node IDs and assert on extracted tokens, so topic extraction is driven by node IDs rather than belief body content.
- `topic-tokenizer-splits-and-filters` — The tests require splitting on `-`, `.`, `:`, and `_`, filtering stop words such as `the`/`use`, and excluding two-character tokens such as `ab`/`cd`.
- `cli-topics-json-output-is-parseable-json` — Running `reasons topics --json` must emit stdout that `json.loads` can parse and that contains `topics` and `total_nodes`.
- `run-cli-catches-systemexit-only` — The `run_cli` helper catches `SystemExit` and maps `None` to exit code `0`, while any non-`SystemExit` exception propagates to pytest.
