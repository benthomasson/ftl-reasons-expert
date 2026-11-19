---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_report_belief.py

# Explanation of `tests/test_report_belief.py`

## Purpose

This file is the test suite for the **belief report** feature. It verifies three layers of the report path:

1. The private markdown/structured-report builder: `_build_structured_report`
2. The public API entry point: `api.report_belief`
3. The CLI command: `reasons report <belief-id>`, including `--help` and `-o/--output`

The tests cover both happy paths and selected edge cases: premise-only beliefs, derived beliefs, `OUT` truth values, outlists, missing beliefs, source-evidence enrichment, and writing a report to a file.

---

## Key Components

### `run_cli(*args, db_path=None)`

A local helper that simulates command-line invocation without spawning a subprocess.

Contract:

- Builds an argv list beginning with `["reasons"]`
- If `db_path` is provided, inserts `["--db", db_path]` before the command arguments
- Patches:
  - `sys.argv`
  - `sys.stdout`
  - `sys.stderr`
- Calls `reasons.cli.main()`
- Catches `SystemExit` and returns `(stdout, stderr, exit_code)`
- If no `SystemExit` is raised, returns `(stdout, stderr, 0)`

This mirrors how real CLI invocation works while keeping assertions simple and in-process.

---

### `TestBuildStructuredReport`

Tests `_build_structured_report`, imported directly from `reasons.report_belief`.

The inferred contract is roughly:

```python
_build_structured_report(
    belief_id,
    node_detail,
    explain_steps,
    premises_data,
    source_chunks,
) -> str
```

The tests establish these input shapes:

- `node_detail`: `{"text": ..., "truth_value": ...}`
- `explain_steps`: list of dictionaries containing:
  - `node`
  - `truth_value`
  - `reason`
  - optionally `antecedents`, `outlist`, `label`
- `premises_data`: list of dictionaries with:
  - `id`
  - `text`
  - `truth_value`
  - `source`
- `source_chunks`: mapping from premise ID to a list of source chunk dictionaries:
  - `filename`
  - `section`
  - `text`
  - `cluster`

Specific tests:

- `test_basic_report`: verifies the markdown heading, belief text, premise IDs, premise text, source, and `"Root Premises (2)"`.
- `test_with_source_chunks`: verifies `"Source Evidence:"`, `"doc.md > Chapter 1"`, and the chunk body.
- `test_no_premises`: verifies `"Root Premises (0)"` and `"No root premises found."`
- `test_out_belief`: verifies `OUT` beliefs render `"status: OUT"` and a `[-]` marker.
- `test_outlist_shown`: verifies outlist identifiers such as `` `blocker-1` `` appear in the report and that the word `"outlist"` is present.

These tests are substring-based, not golden-file based. The report format can evolve as long as these important markers remain.

---

### `TestApiReportBelief`

Tests the public API function `reasons.api.report_belief`.

Setup typically uses:

```python
api.add_node(id, text, db_path=db)
api.add_node(id, text, sl="premise-a,premise-b", db_path=db)
```

The inferred contract of `api.report_belief` is:

```python
report_belief(belief_id, db_path=..., sources_db=...) -> dict
```

The returned dictionary must contain at least:

- `"premise_count"`: integer
- `"report"`: rendered markdown string

Tests:

- `test_basic_report`: creates two premises and one derived belief, then checks `premise_count == 2` and that all IDs appear in the report.
- `test_premise_report`: reports a premise directly; checks `premise_count == 1`.
- `test_with_sources_db`: creates a separate SQLite `sources.db` with a `chunks` table and `chunks_fts` FTS5 table, then checks that source evidence is incorporated or at least the source text appears.
- `test_missing_node_raises`: initializes an empty DB and expects `KeyError` when reporting a nonexistent belief.

The source DB schema used by the test is:

```sql
CREATE TABLE chunks (
    id INTEGER PRIMARY KEY,
    text TEXT,
    cluster TEXT,
    filename TEXT,
    section TEXT
);

CREATE VIRTUAL TABLE chunks_fts USING fts5(
    text,
    content=chunks,
    content_rowid=id
);
```

The test inserts one chunk and one FTS row, implying the production code searches source chunks via FTS and maps them back to premise IDs.

---

### `TestCliReport`

Tests the CLI command via `run_cli`.

Tests:

- `test_help`: runs `reasons report --help`, expects exit code `0` and `report` in lowercased stdout.
- `test_basic`: creates `p1` and derived `d1`, runs `reasons report d1`, expects exit code `0` and output containing `"Belief Report"`, `"d1"`, and `"p1"`.
- `test_output_file`: runs `reasons report p1 -o <path>`, expects exit code `0` and the file to contain `"Belief Report"`.

The CLI argument order in these tests is:

```text
reasons --db <db_path> report <belief-id> [-o <output-file>]
```

---

## Patterns

- **Layered testing**: direct helper test, public API integration test, CLI end-to-end test.
- **Pytest class grouping**: tests are grouped by component under `TestBuildStructuredReport`, `TestApiReportBelief`, and `TestCliReport`.
- **Temporary filesystem isolation**: `tmp_path` is used for both belief DBs and source DBs.
- **Monkeypatching**: `unittest.mock.patch` replaces `sys.argv`, `sys.stdout`, and `sys.stderr` for CLI tests.
- **Real SQLite usage**: the API test constructs a real SQLite source database rather than mocking storage.
- **Substring assertions**: markdown output is checked for important fragments rather than exact full-document equality.
- **Arrange-Act-Assert**: each test seeds data, invokes one operation, and asserts on returned output.

A small maintenance note: `import json` is present but unused in this file.

---

## Dependencies

### Production modules used

- `reasons.api`
  - `api.add_node(...)`
  - `api.init_db(...)`
  - `api.report_belief(...)`
- `reasons.report_belief._build_structured_report`
- `reasons.cli.main`

### Standard library

- `sqlite3` for constructing the source-evidence database
- `sys` for argv/stdout/stderr patching
- `io.StringIO` for capturing CLI output
- `unittest.mock.patch`
- `pytest` for `pytest.raises` and test discovery

### Imported by

This is a pytest test module. It is not imported by production code. Pytest discovers it by filename convention and executes the test classes/functions.

The direct import of `_build_structured_report` means the test suite pins the private symbol name in `reasons.report_belief`.

---

## Flow

### 1. Structured report unit flow

```text
test builds node_detail + explain_steps + premises_data + source_chunks
    -> calls _build_structured_report(...)
    -> receives markdown string
    -> asserts important substrings exist
```

### 2. API flow

```text
tmp_path creates DB path
    -> api.add_node(...) seeds premises/derived nodes
    -> optionally builds sources.db with chunks + chunks_fts
    -> api.report_belief(id, db_path=..., sources_db=...)
    -> returns {"premise_count": int, "report": str}
    -> test checks count and report content
```

### 3. CLI flow

```text
run_cli("report", "d1", db_path=db)
    -> builds argv ["reasons", "--db", db, "report", "d1"]
    -> patches sys.argv/stdout/stderr
    -> calls reasons.cli.main()
    -> main parses args, invokes report logic, prints/writes output
    -> SystemExit is caught
    -> returns captured stdout/stderr/exit code
    -> test asserts stdout or output-file content
```

---

## Invariants

- `_build_structured_report` must emit a top-level heading in the form:
  - `# Belief Report: \`<belief-id>\``
- The report must include the belief text and status/truth value.
- Root-premise count must render as:
  - `Root Premises (N)`
- When there are no premises, the report must include:
  - `No root premises found.`
- `OUT` beliefs must render status text including `status: OUT` and a `[-]` marker.
- Outlist entries must be visible in the report and the report must mention `outlist`.
- Source evidence must render section paths as:
  - `<filename> > <section>`
- `api.report_belief` must return a dictionary with `premise_count` and `report`.
- Reporting a missing belief must raise `KeyError`.
- Successful CLI invocations must exit with code `0`.
- The CLI helper treats absence of `SystemExit` as exit code `0`.

---

## Error Handling

- `api.report_belief("nonexistent", db_path=db)` is expected to raise `KeyError`. The test does not check the exception message.
- `run_cli` catches `SystemExit`, which is how `argparse` handles `--help` and CLI exits.
- `run_cli` does **not** catch ordinary exceptions. A non-`SystemExit` failure inside `main()` will fail the test directly.
- The source DB setup manually opens and closes a SQLite connection. There is no `try/finally` around it, but the test is short and single-purpose.
- The tests do not assert stderr contents or invalid CLI error paths beyond `--help`.

---

## Topics to Explore

- [file] `reasons/report_belief.py` — Implements `_build_structured_report` and the report generation logic exercised by the first test class.
- [function] `reasons/api.py:report_belief` — Public API contract, premise counting, missing-node behavior, and source DB integration.
- [file] `reasons/cli.py` — CLI argument wiring for `report`, `--db`, and `-o`, plus `SystemExit` behavior.
- [general] `source-evidence-schema` — How `chunks` and `chunks_fts` are populated and queried to attach source evidence to premises.
- [file] `tests/test_report_gated.py` — Sibling tests likely covering gated/outlist report behavior from another angle.

## Beliefs

- `report-belief-returns-report-and-premise-count` — `api.report_belief` returns a dictionary containing a `report` string and an integer `premise_count`.
- `missing-belief-raises-keyerror` — `api.report_belief` raises `KeyError` for a nonexistent belief ID.
- `cli-runner-catches-systemexit` — `run_cli` catches `SystemExit` from `reasons.cli.main` and returns its exit code with captured stdout/stderr.
- `structured-report-includes-root-premises-and-source-evidence` — `_build_structured_report` includes root-premise counts and source-evidence sections when those inputs are provided.
- `outlist-rendered-in-report` — Outlist identifiers from explanation steps are rendered into the generated belief report.
