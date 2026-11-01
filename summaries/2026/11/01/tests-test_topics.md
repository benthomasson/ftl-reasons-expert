---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_topics.py

# `tests/test_topics.py` — Topics Command Test Suite

## 1. Purpose

This file is the integration test suite for the **topics extraction feature** of the `reasons` system. The "topics" command analyzes node IDs in the belief graph, tokenizes them, filters noise, and returns a frequency-ranked list of recurring topic words. This file validates both:

- The **API contract** (`api.topics()`) — data shapes, filtering rules, sorting, limits
- The **CLI surface** (`reasons topics`) — argument parsing, human-readable output, `--json` mode, exit codes

It owns the responsibility of proving that topic extraction behaves correctly from raw node identifiers through to the final user-facing output.

## 2. Key Components

### `run_cli(*args, db_path=None)` (module-level helper, line 13)

| Aspect | Detail |
|--------|--------|
| **Signature** | `run_cli(*args, db_path=None) → (str, str, int)` |
| **Contract** | Spawns `reasons.cli.main()` in-process with mocked `sys.argv`, `sys.stdout`, `sys.stderr`. Returns captured stdout, stderr, and the process exit code. |
| **Key behavior** | Catches `SystemExit` (which `main()` raises on exit) and normalizes `e.code is None` → `0`. This means a "clean" exit always yields code 0. |

### `TestTopicsApi` (line 30)

Tests the `api.topics()` function directly. Each test creates a fresh SQLite DB via `tmp_path`, seeds nodes with `api.add_node()`, calls `api.topics()`, and asserts on the returned dict.

| Test | What it verifies |
|------|-----------------|
| `test_empty_db` | Empty graph → `{"topics": [], "total_nodes": 0}` |
| `test_extracts_words_from_ids` | Hyphen-separated IDs are split; expected words appear; per-topic counts are correct |
| `test_filters_stop_words` | Common English words ("the", "use") are excluded |
| `test_filters_short_words` | Tokens ≤ 2 chars are excluded |
| `test_splits_on_delimiters` | Dots, colons, underscores are all valid delimiters (e.g., `module.class:method_name`) |
| `test_limit` | `limit=5` caps the result list length |
| `test_sorted_by_frequency` | Most-frequent topic appears first; its `count` reflects all occurrences |
| `test_total_nodes` | `total_nodes` equals the number of nodes in the graph |

### `TestCmdTopics` (line 112)

Tests the CLI command end-to-end via `run_cli()`.

| Test | What it verifies |
|------|-----------------|
| `test_help` | `topics --help` exits 0 and prints "topics" |
| `test_basic_output` | Human-readable output contains topic names and the phrase "topics from" |
| `test_json_output` | `--json` produces valid JSON with `topics` and `total_nodes` keys |
| `test_limit_flag` | `--limit 3` produces output stating "3 topics from" |
| `test_empty_db` | Empty DB prints "No topics found" and exits 0 |

## 3. Patterns

- **In-process CLI testing**: Instead of spawning a subprocess, `run_cli()` patches `sys.argv`/`sys.stdout`/`sys.stderr` and calls `main()` directly. This is faster and avoids file-descriptor leaks, but couples tests to the `reasons.cli.main()` entry point.
- **`tmp_path` isolation**: Every test gets a unique temp directory (provided by pytest), so no test shares state with another. The DB is created at `tmp_path / "test.db"`.
- **Two-layer testing**: API tests verify data contracts; CLI tests verify presentation and argument wiring. A regression in one layer won't be masked by the other.
- **`patch.object` + context manager**: The mock patches are scoped to the `with` block, guaranteeing restoration even on exceptions.
- **`SystemExit` as control flow**: `main()` uses `sys.exit(n)` (via `SystemExit`) rather than returning a code. The helper converts this to a tuple.

## 4. Dependencies

**This file imports:**
- `json` — parsing `--json` CLI output
- `sys`, `io.StringIO`, `unittest.mock.patch` — CLI mocking infrastructure
- `pytest` — test runner, `tmp_path` fixture
- `reasons.api` — the function under test (`topics`, `add_node`)
- `reasons.cli` — the CLI entry point (`main`)

**What depends on this file:**
- Nothing in the source tree imports from `tests/`. It is a leaf in the dependency graph, consumed only by the test runner (pytest).
- The `tests/conftest.py` likely provides shared fixtures, but `tmp_path` is built-in pytest.

## 5. Flow

```
Test invocation (pytest)
│
├─ TestTopicsApi.test_*
│   1. db = str(tmp_path / "test.db")
│   2. api.add_node(id, content, db_path=db)   ← one or more calls
│   3. result = api.topics(db_path=db, [limit=N])
│   4. assert on result["topics"] list shape, result["total_nodes"]
│
└─ TestCmdTopics.test_*
    1. db = str(tmp_path / "test.db")
    2. api.add_node(...)                        ← seed data
    3. run_cli("topics", ["--json" | "--limit", "3"], db_path=db)
         │
         ├─ build argv = ["reasons", "--db", db, "topics", ...]
         ├─ patch sys.argv / sys.stdout / sys.stderr
         ├─ call reasons.cli.main()
         │     ├─ argparse parses "topics" subcommand
         │     ├─ calls api.topics(...)
         │     └─ prints human text or JSON to stdout
         ├─ catch SystemExit → extract code
         └─ return (stdout_str, stderr_str, code)
    4. assert on stdout string / parsed JSON / exit code
```

Data transformation chain: **node ID string** → tokenize (split on `-`, `.`, `:`, `_`) → filter (stop-words, length < 3) → count frequencies → sort descending → truncate to limit → wrap in `{"topics": [...], "total_nodes": N}`.

## 6. Invariants

1. **Token length floor**: Words with fewer than 3 characters are never included in topics (enforced by `test_filters_short_words`).
2. **Stop-word exclusion**: A curated stop-word list removes common English function words (enforced by `test_filters_stop_words`).
3. **Delimiter set**: The tokenizer recognizes at minimum `-`, `.`, `:`, and `_` as word boundaries (enforced by `test_splits_on_delimiters`).
4. **Sort order**: Topics are always sorted by descending frequency; ties presumably broken alphabetically (implied by `test_sorted_by_frequency`).
5. **Shape contract**: `api.topics()` always returns a dict with exactly the keys `"topics"` (list of `{"topic": str, "count": int}`) and `"total_nodes"` (int).
6. **Exit code semantics**: A successful CLI invocation always exits 0, even on empty data. Non-zero is reserved for genuine failures.
7. **`total_nodes` reflects the full graph**, not just the nodes contributing to the returned (possibly truncated) topic list.

## 7. Error Handling

- **`SystemExit` swallowing**: `run_cli()` deliberately catches `SystemExit` and converts it to an integer code. This means the test harness never lets an exception propagate out of the `with` block — it's treated as a normal return path.
- **`e.code is None` → 0**: If `sys.exit()` is called without an argument, the code is normalized to 0.
- **No explicit error-path tests**: There are no tests for, e.g., a corrupt DB, a missing `--db` flag, or an invalid `--limit` value. Error handling for those paths (if any) is not covered here.
- **Empty data is not an error**: Both API and CLI treat an empty graph as a valid state (return empty list / print "No topics found", exit 0).

---

## Topics to Explore

- [function] `reasons/api.py:topics` — The actual implementation: how tokenization, stop-word filtering, and frequency counting are performed; what the exact delimiter regex and stop-word list are.
- [file] `reasons/cli.py` — How the `topics` subcommand is registered, what flags it accepts (`--json`, `--limit`, `--db`), and how it formats the human-readable output string.
- [file] `tests/conftest.py` — Whether any shared fixtures or DB-creation helpers are defined here that these tests rely on implicitly.
- [general] `stop-word-list-source` — Whether the stop-word list is hardcoded in `api.py`, loaded from a data file, or configurable via environment/CLI flag; affects maintainability.
- [file] `reasons_lib/api.py` — The `reasons_lib` mirror of the same modules; understanding whether tests exercise `reasons` or `reasons_lib` and how the two relate (vendoring? re-export?).

---

## Beliefs

- `test-topics-exercises-api-and-cli` — This file tests two distinct layers (`api.topics()` and `reasons topics` CLI); a regression in the CLI formatting that doesn't affect the API return shape will only be caught by `TestCmdTopics`, and vice versa.
- `topics-tokenizer-splits-on-four-delimiters` — The tokenizer recognizes `-`, `.`, `:`, and `_` as word boundaries; any new delimiter added to the implementation must be covered by a corresponding test in `test_splits_on_delimiters`.
- `topics-result-shape-is-total-and-topics` — `api.topics()` always returns exactly two top-level keys (`"topics"`, `"total_nodes"`); adding a new key (e.g., `"elapsed_ms"`) would break any downstream consumer doing strict key validation like the `--json` CLI path.
- `run-cli-normalizes-none-exit-to-zero` — `run_cli()` maps `SystemExit(code=None)` to exit code 0, so a test asserting `code == 0` cannot distinguish a successful run from a bare `sys.exit()` call; only non-zero codes signal failure.
- `tmp-path-isolates-each-test` — Every test in this file uses the pytest `tmp_path` fixture, guaranteeing no shared DB state between tests; any test that writes to a fixed path instead would introduce ordering dependencies.
