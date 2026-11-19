---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_report_gated.py

# Explanation of `tests/test_report_gated.py`

## Purpose

This is a pytest regression suite for the **report-gated** feature: generating a Markdown report about beliefs blocked by active `unless`/blocker nodes and about retracted premises (“fixed defects”). It tests four layers:

1. Pure Markdown report formatting via `_build_structured_report`.
2. Prompt/data formatting via `_format_data_for_prompt`.
3. The public API facade via `api.report_gated(...)`.
4. LLM and CLI integration, including model passthrough and `-o` file output.

The tests own the stable output contract of the report: title, summary counts, blocker/retracted sections, sorting, empty-state behavior, LLM behavior, and CLI stream/file behavior.

---

## Key Components

### `run_cli(*args, db_path=None)`

A local helper that invokes the real CLI in-process.

- Builds argv as:
  - `["reasons"]`
  - optionally `["--db", db_path]`
  - then the provided CLI args.
- Patches `sys.argv`, `sys.stdout`, and `sys.stderr`.
- Calls `reasons.cli.main()`.
- Catches `SystemExit` and returns its code, normalizing `None` to `0`.
- Returns `(stdout, stderr, code)`.

Important detail: `--db` is inserted **before** the subcommand, e.g.:

```python
["reasons", "--db", db_path, "report-gated"]
```

so the test enforces that `--db` is a global CLI option accepted before `report-gated`.

Also, `run_cli` ignores the return value of `main()`. It assumes success means `main()` returns normally or raises `SystemExit(0)`.

---

### `db(tmp_path)` fixture

Creates an isolated SQLite database using `reasons.api` calls.

Nodes created:

- `normal-premise-a`, `normal-premise-b`: ordinary IN premises.
- `blocker-x`: a node acting as a blocker.
- `support-for-gated`: support for the gated node.
- `gated-conclusion`: created with:
  - `sl="support-for-gated"`
  - `unless="blocker-x"`
- `retracted-defect`: later retracted with reason `"Fixed in PR #99"`.

The fixture is designed so `api.report_gated(db_path=db)` should report:

- `blocker_count == 1`
- `gated_count == 1`
- `retracted_count == 1`

It exercises the real storage/API layer rather than mocking database data.

---

### `TestBuildStructuredReport`

Tests the private Markdown builder:

```python
_build_structured_report(in_count, out_count, gated, retracted, details)
```

The synthetic `gated` shape is:

```python
{
    "blockers": {
        "blocker-id": {
            "text": "...",
            "gated": [{"id": "...", "text": "..."}],
        },
    },
    "blocker_count": int,
    "gated_count": int,
}
```

Tests cover:

- Report title: `# Gated Beliefs Report`
- Summary line: `100 IN / 50 OUT`
- Active blocker count: `1 active blocker(s)`
- Retracted/fixed-defect section containing:
  - `Fixed Defects`
  - backticked ID
  - retraction reason
  - dependent count
- Active blocker rendering:
  - backticked blocker ID
  - blocker text
  - `Gates 1 belief(s)`
  - gated node ID
- Empty network placeholder:
  - `No blockers or retracted premises found.`
- No `Fixed Defects` section when there are no retracted premises.
- Blockers sorted by number of gated beliefs descending:
  - `blocker-many` must appear before `blocker-few`.

The helper `_make_gated_data()` builds default synthetic data for this class.

---

### `TestFormatDataForPrompt`

Tests:

```python
_format_data_for_prompt(in_count, out_count, gated, retracted, details)
```

This is likely the plain-text/LLM prompt input formatter. Tests verify it includes:

- IN/OUT counts: `50 IN`, `20 OUT`
- Retracted node ID, text, and reason.
- Blocker ID, blocker text, and gated node ID.

Unlike the structured report, these tests do not require Markdown formatting.

---

### `TestReportGatedApi`

Tests the public API facade:

```python
api.report_gated(db_path=db)
```

The return value is a dict containing at least:

- `report`: Markdown report string
- `blocker_count`
- `gated_count`
- `retracted_count`

Tests verify:

- Report string contains `# Gated Beliefs Report`.
- Counts match the fixture.
- Report contains `blocker-x` and `gated-conclusion`.
- Report contains `retracted-defect` and `Fixed in PR #99`.
- Empty DB returns zero counts and still includes the report title.

---

### `TestReportGatedLlm`

Tests the lower-level orchestration function:

```python
report_gated(gated, retracted, details, in_count, out_count, model=...)
```

Note the argument order differs from the formatting helpers: counts come after the data arguments.

Tests patch:

```python
reasons.llm.invoke_model
```

and verify:

- When `model="claude"` is supplied, `invoke_model` is called exactly once.
- The LLM output appears in the returned report.
- When no model is supplied, `invoke_model` is not called.
- LLM mode still includes deterministic title and summary stats:
  - `# Gated Beliefs Report`
  - `10 IN / 5 OUT`
- `api.report_gated(model="claude", db_path=db)` forwards the model to the lower-level function and invokes the model exactly once.

A key implementation dependency: `report_gated` must resolve `invoke_model` through the `reasons.llm` module attribute at call time, otherwise patching `reasons.llm.invoke_model` would not intercept it.

---

### `TestReportGatedCli`

Tests the `report-gated` CLI command.

- `--help` exits with code `0` and prints help text mentioning `report-gated` or `report`.
- Default invocation writes the Markdown report to **stdout**.
- Default invocation writes a summary containing `blocker(s)` to **stderr**.
- With `-o out_file`, the CLI:
  - writes `Report written to` to stdout,
  - creates the file,
  - writes a report containing `# Gated Beliefs Report`.

This confirms the CLI separates machine-readable/report content from human-facing status output.

---

## Patterns

- **Layered test coverage**: pure formatting tests, API integration tests, LLM-mocking tests, and CLI integration tests.
- **Fixture-based DB isolation**: `tmp_path` gives each test a fresh SQLite database path.
- **Synthetic data builders**: `_make_gated_data()` avoids needing a real DB for formatting tests.
- **Mocking external LLM calls**: `unittest.mock.patch("reasons.llm.invoke_model", ...)` keeps tests deterministic and offline.
- **CLI harness pattern**: `run_cli()` centralizes argv/stdout/stderr patching and `SystemExit` handling.
- **Contract testing of private helpers**: tests directly import `_build_structured_report` and `_format_data_for_prompt`, so their names/signatures are effectively part of the tested surface even though they are private.

---

## Dependencies

### Direct test dependencies

- `os`: used for `os.path.isfile`.
- `sys`, `StringIO`, `patch`: CLI capture and mocking.
- `pytest`: fixtures and test discovery.
- `reasons.api`: real API facade for DB-backed tests.
- `reasons.report_gated`:
  - `_build_structured_report`
  - `_format_data_for_prompt`
  - `report_gated`
- `reasons.cli.main`: imported inside `run_cli`.

### Indirect production dependencies exercised

- `reasons.storage` / SQLite via `api.add_node`, `api.retract_node`, `api.report_gated`.
- `reasons.llm.invoke_model` via patching.
- CLI command registration and argument parsing in `reasons.cli`.

### Imported by

This test file is not imported by production code. It is discovered by pytest.

Repository note: the repo contains both `reasons/` and `reasons_lib/` with parallel files. These tests import `reasons`, so they target the `reasons` package, not `reasons_lib`.

---

## Flow

1. Formatting tests synthesize `gated`, `retracted`, and `details` dicts and call `_build_structured_report` or `_format_data_for_prompt` directly.
2. API tests use the `db` fixture:
   - create nodes via `api.add_node`,
   - retract one node via `api.retract_node`,
   - call `api.report_gated(db_path=db)`,
   - assert on the returned dict.
3. LLM tests patch `reasons.llm.invoke_model`, call `report_gated(...)`, and assert model invocation count and returned content.
4. CLI tests call `run_cli(...)`:
   - `run_cli` constructs argv,
   - patches `sys.argv`/stdout/stderr,
   - calls `reasons.cli.main()`,
   - captures output and exit status.
5. For file output, the CLI test passes `-o <tmp_path>/report.md` and verifies both stdout status and file contents.

---

## Invariants

- Every structured/API/CLI report starts with `# Gated Beliefs Report`.
- Summary stats include `<in_count> IN / <out_count> OUT`.
- Active blocker summary includes `<n> active blocker(s)`.
- When there are no blockers and no retracted premises, the report contains:
  - `No blockers or retracted premises found.`
- `Fixed Defects` is omitted when there are no retracted premises.
- Retracted entries render backticked IDs, retraction reason, and dependent count.
- Active blockers render backticked IDs, blocker text, `Gates N belief(s)`, and gated IDs.
- Blockers are sorted by gated-belief count descending.
- `report_gated(..., model=...)` calls the LLM exactly once and includes its output.
- `report_gated(..., model=None)` does not call the LLM.
- LLM output does not replace deterministic title and stats.
- `api.report_gated` returns a dict with `report`, `blocker_count`, `gated_count`, and `retracted_count`.
- CLI default output sends the report to stdout and a `blocker(s)` summary to stderr.
- `-o` writes the report to the requested file and prints `Report written to` to stdout.

---

## Error Handling

- `run_cli` catches only `SystemExit`. Any other exception raised by `main()` propagates and fails the test.
- `SystemExit.code is None` is normalized to `0`.
- `run_cli` does not inspect a returned integer from `main()`. If `main()` returned a non-zero value without raising, `run_cli` would still report `0`.
- No tests cover:
  - invalid DB paths,
  - missing output directories,
  - LLM failures,
  - CLI non-zero error exits,
  - malformed `gated`/`retracted` data.
- The file-output test opens the file with `open(out_file).read()` without a context manager; this is harmless in the test but not a general resource-handling pattern.

---

## Topics to Explore

- [file] `reasons/report_gated.py` — Contains the actual report builder, prompt formatter, and LLM orchestration logic tested here.
- [function] `reasons/report_gated.py:_build_structured_report` — Implements the Markdown layout, sorting, and empty-state behavior.
- [function] `reasons/api.py:report_gated` — Public API facade that the DB-backed tests exercise.
- [file] `reasons/cli.py` — Defines `report-gated`, global `--db`, `-o`, stdout/stderr behavior, and `main()`.
- [general] `reasons-vs-reasons_lib` — The repo appears to contain mirrored packages; tests target `reasons`, so changes may need synchronization with `reasons_lib`.

---

## Beliefs

- `report-gated-report-has-markdown-title` — All tested report outputs include the exact string `# Gated Beliefs Report`.
- `report-gated-blockers-sorted-by-gated-count-descending` — `_build_structured_report` must list a blocker with more gated beliefs before one with fewer.
- `report-gated-llm-invocation-is-patchable-through-reasons-llm` — Tests patch `reasons.llm.invoke_model` and require exactly one call when `model` is provided.
- `api-report-gated-returns-report-and-counts` — `api.report_gated` returns a dict containing `report`, `blocker_count`, `gated_count`, and `retracted_count`.
- `cli-report-gated-splits-report-and-summary-streams` — The default CLI writes the Markdown report to stdout and a `blocker(s)` summary to stderr.
