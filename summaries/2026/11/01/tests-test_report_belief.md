---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_report_belief.py

# `tests/test_report_belief.py`

## Purpose

This file is the test suite for the **belief report** feature — the ability to generate a human-readable Markdown report explaining *why* a node in the belief network is true (IN) or false (OUT), including its derivation chain, root premises, and source evidence.

It covers three layers of the `report_belief` functionality:

1. **Unit level** — `_build_structured_report`, the internal function that assembles the Markdown string from raw data structures.
2. **API level** — `api.report_belief`, the public function that queries the database, gathers explain steps, premises, and source chunks, then delegates to the builder.
3. **CLI level** — the `reasons report <node>` command as invoked through `reasons.cli.main`.

The file owns the contract: given a belief network in a SQLite database, the report must faithfully represent the node's status, its justification chain, its root premises (with text and source), and any outlist blockers.

## Key Components

### `run_cli(*args, db_path=None)` — helper

```python
def run_cli(*args, db_path=None):
    argv = ["reasons"]
    if db_path:
        argv += ["--db", db_path]
    argv += list(args)
    stdout, stderr = StringIO(), StringIO()
    with patch.object(sys, "argv", argv), \
         patch.object(sys, "stdout", stdout), \
         patch.object(sys, "stderr", stderr):
        try:
            main()
        except SystemExit as e:
            return stdout.getvalue(), stderr.getvalue(), e.code
    return stdout.getvalue(), stderr.getvalue(), 0
```

- **Contract**: Simulates a full CLI invocation without spawning a subprocess. Returns a `(stdout, stderr, exit_code)` tuple.
- Patches `sys.argv`, `sys.stdout`, `sys.stderr` simultaneously.
- Catches `SystemExit` (which `click` or `argparse` raise) to extract the exit code. If `main()` returns normally, code defaults to `0`.
- All CLI tests in this file use this helper — there is no subprocess-based testing here.

### `TestBuildStructuredReport`

Tests the **pure function** `_build_structured_report(node_id, node_detail, explain_steps, premises_data, source_chunks)`. No database is involved.

| Test | What it verifies |
|---|---|
| `test_basic_report` | A derived node with two premises produces a report containing the node ID, its text, both premise IDs, premise texts, source filenames, and a `Root Premises (2)` heading. |
| `test_with_source_chunks` | When `source_chunks` is populated, the report includes a `Source Evidence:` section with `filename > section` breadcrumbs and the chunk text. |
| `test_no_premises` | A premise node with no further premises shows `Root Premises (0)` and the message `No root premises found.` |
| `test_out_belief` | An OUT node shows `status: OUT` and a `[-]` marker (visual indicator of retraction). |
| `test_outlist_shown` | When an explain step carries an `outlist` key, the blocker IDs appear in the report alongside the word `outlist`. |

### `TestApiReportBelief`

Integration tests that **create a real SQLite database** via the `tmp_path` fixture, populate it with `api.add_node`, then call `api.report_belief`.

| Test | What it verifies |
|---|---|
| `test_basic_report` | A 2-premise derivation returns `premise_count == 2` and a report string containing all three node IDs. |
| `test_premise_report` | A single premise node reports `premise_count == 1` and includes its own ID. |
| `test_with_sources_db` | Creates a **second** SQLite DB (`sources.db`) with a `chunks` table + FTS5 index, inserts a chunk referencing `PostgreSQL`, then verifies the report includes source evidence. This is the only test that exercises the `sources_db` parameter. |
| `test_missing_node_raises` | Calls `api.report_belief("nonexistent")` on an empty DB and asserts a `KeyError` is raised. |

### `TestCliReport`

End-to-end tests through `reasons.cli.main`.

| Test | What it verifies |
|---|---|
| `test_help` | `reasons report --help` exits 0 and prints "report". |
| `test_basic` | `reasons report d1` prints a `Belief Report` containing both `d1` and `p1`. |
| `test_output_file` | `reasons report p1 -o <file>` writes the report to disk; stdout assertion is skipped in favor of reading the file. |

## Patterns

- **Three-layer test pyramid**: unit (pure function) → integration (API + real DB) → CLI (full entry point). Each layer tests progressively more of the stack.
- **`tmp_path` isolation**: Every test that touches the database uses pytest's `tmp_path` fixture, so tests are hermetic and leave no artifacts.
- **CLI simulation via `patch`**: Rather than using `subprocess.run`, the `run_cli` helper patches `sys.argv`/`sys.stdout`/`sys.stderr` in-process. This is faster and avoids PATH/env issues, but it means the test shares the process with the code under test — any global state leaks.
- **String-contains assertions**: The tests assert on substrings of the Markdown report (e.g. `"Root Premises (2)" in report`). This is brittle to formatting changes but avoids depending on a Markdown parser.
- **Direct test of a private function**: `TestBuildStructuredReport` imports `_build_structured_report` (underscore-prefixed, i.e. "private"). This is a deliberate choice — the report-building logic is a pure function and is tested in isolation to give fast, precise failure messages. If you refactor the API to pass different data shapes, this class will break first.

## Dependencies

### What this file imports (upstream)

| Import | Role |
|---|---|
| `reasons.api` | Public API: `add_node`, `report_belief`, `init_db`. The primary SUT for the integration tests. |
| `reasons.report_belief._build_structured_report` | The internal Markdown assembler. SUT for the unit tests. |
| `reasons.cli.main` | CLI entry point. SUT for the CLI tests. |
| `pytest` | Test runner + `tmp_path` fixture + `pytest.raises`. |
| `unittest.mock.patch` | In-process patching of `sys.argv`, `sys.stdout`, `sys.stderr`. |
| `sqlite3` | Used directly in `test_with_sources_db` to create the FTS5 sources database. |
| `io.StringIO` | In-memory streams for captured CLI output. |
| `json`, `sys` | `json` is imported but **unused** in this file — likely a leftover. |

### What depends on this file (downstream)

None. This is a leaf test module. No other module imports from `tests/`.

## Flow

```
Test setup (all integration/CLI tests)
  │
  ├─ tmp_path / "test.db"  ←  SQLite file created by api.init_db / api.add_node
  │
  └─ (test_with_sources_db only)
       tmp_path / "sources.db"  ←  sqlite3.create + FTS5 + INSERT chunk

Test execution
  │
  ├─ TestBuildStructuredReport
  │    │  Constructs dicts in-memory (node_detail, explain_steps, premises_data, source_chunks)
  │    └─ calls _build_structured_report(...) → returns a Markdown string
  │    └─ asserts substrings in that string
  │
  ├─ TestApiReportBelief
  │    │  api.add_node(id, text, sl=..., db_path=db)   ←  writes to SQLite
  │    └─ api.report_belief(node_id, db_path=db, [sources_db=...])
  │         internally: reads node row → walks explain chain → gathers premises
  │         → queries sources_db for chunks → calls _build_structured_report
  │         → returns {"report": str, "premise_count": int, ...}
  │    └─ asserts on result["premise_count"] and result["report"]
  │
  └─ TestCliReport
       │  run_cli("report", node_id, db_path=db, [-o file])
       │    patches sys.argv = ["reasons", "--db", db, "report", node_id]
       │    patches sys.stdout / sys.stderr to StringIO
       │    calls main()  ←  argparse/click parses, calls api.report_belief, prints
       │    catches SystemExit → extracts code
       └─ asserts on captured stdout or file contents
```

The data transformation chain for a single report is:

**SQLite row → `node_detail` dict → `explain_steps` list → `premises_data` list → `source_chunks` dict → `_build_structured_report` → Markdown string → (CLI) stdout or file**

## Invariants

1. **Report always contains the target node's ID.** Every test asserts the node ID appears in the report (e.g. `"derived-boil" in report`). The report heading is `# Belief Report: \`<id>\``.
2. **Premise count is exact.** `result["premise_count"]` must equal the number of leaf premises reachable through the `sl` justification chain. The report heading `Root Premises (N)` must match.
3. **OUT nodes are visually distinguished.** The report must include `status: OUT` and a `[-]` marker. If an `outlist` is present, each blocker ID must be listed.
4. **Missing nodes are a hard error.** `api.report_belief` raises `KeyError` for a non-existent node ID — it does not return an empty report or a dict with an error field.
5. **Source evidence is optional but structured.** When `source_chunks` is non-empty, the report includes a `Source Evidence:` section with `filename > section` breadcrumbs. When empty, that section is absent (implicit from `test_no_premises` and `test_basic_report` not asserting its presence).

## Error Handling

- **`KeyError` for missing nodes**: The only explicitly tested error path is `test_missing_node_raises`, which uses `pytest.raises(KeyError)`. This tells you that `api.report_belief` looks up the node and raises if absent — it does **not** return a sentinel or log-and-continue.
- **CLI exit codes**: `run_cli` returns `e.code` from `SystemExit`. All tested CLI paths return `0`. There is no test for a CLI-level error (e.g. running `reasons report` with a non-existent node) — that path is untested.
- **No exception swallowing**: The tests do not wrap calls in try/except (except `run_cli` catching `SystemExit`, which is a control-flow mechanism, not error suppression). If `api.report_belief` raised an unexpected exception, pytest would report it as a test failure.
- **Unused import**: `json` is imported but never used. This is a minor code-hygiene issue, not a functional concern.

---

## Topics to Explore

- [file] `reasons/report_belief.py` — The implementation under test. Understanding `_build_structured_report`'s exact Markdown template and how it handles edge cases (nested derivations, multiple justifications, empty outlists) will make the test assertions less opaque.
- [file] `reasons/api.py` — The `report_belief` and `add_node` functions. You'll see how the explain-chain walk, premise collection, and source-chunk lookup are orchestrated before handing data to the report builder.
- [file] `reasons/cli.py` — The `report` subcommand definition. Shows how CLI args (`--db`, `-o`, positional node ID) map to the `api.report_belief` call, and where the `SystemExit` that `run_cli` catches originates.
- [function] `reasons/report_belief.py:_build_structured_report` — The pure function that 12 of the 13 tests in this file depend on. Its signature and the shape of `explain_steps` / `premises_data` / `source_chunks` are the contract the tests lock in.
- [file] `tests/conftest.py` — Shared fixtures. While this test file only uses `tmp_path` (built-in), the conftest may define DB fixtures or mocks that other test files rely on, giving context for conventions.

---

## Beliefs

- `report-belief-raises-keyerror-for-missing-node` — `api.report_belief` raises `KeyError` (not a custom exception, not a sentinel return) when the requested node ID does not exist in the database, as verified by `test_missing_node_raises`.
- `test-build-structured-report-tests-private-function` — The test suite imports and tests `_build_structured_report` (underscore-prefixed), making it de-facto part of the public contract; refactoring its signature or splitting it will break 5 unit tests before any integration test runs.
- `run-cli-simulates-in-process` — CLI tests patch `sys.argv`/`sys.stdout`/`sys.stderr` in the same process rather than spawning a subprocess, meaning global state (e.g. module-level singletons, open file handles) from earlier tests in the same session can leak into later CLI tests.
- `source-chunks-require-fts5-index` — The `test_with_sources_db` test creates a `chunks` table **and** a `chunks_fts` FTS5 virtual table; `api.report_belief` likely queries the FTS index to find relevant chunks, so a sources DB missing the FTS table would fail silently or with a DB error.
- `premise-count-reflects-leaf-nodes-only` — `result["premise_count"]` counts only root/leaf premises (nodes with `reason: "premise"` in the explain steps), not intermediate derived nodes, as shown by `test_basic_report` expecting `2` for a 3-node chain.
