---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_report_gated.py

# `tests/test_report_gated.py`

## Purpose

This file is the test suite for the **gated beliefs reporting** feature of `ftl-reasons`. A "gated belief" is a conclusion that *would* be true (IN) except that one or more active "blocker" premises prevent it. The `report_gated` command produces a human-readable (or LLM-narrated) summary of all such gated conclusions, active blockers, and retracted premises in a belief network.

The tests own the contract for three layers:

| Layer | Under-test symbol | What's verified |
|-------|-------------------|-----------------|
| Pure function | `_build_structured_report` | Markdown output formatting, ordering, edge cases |
| Pure function | `_format_data_for_prompt` | Data→prompt string transformation |
| Orchestration | `report_gated` | LLM invocation, report assembly |
| Public API | `api.report_gated` | End-to-end dict return shape |
| CLI | `reasons.cli.main` | `--help`, stdout vs. file output, exit codes |

## Key Components

### `run_cli(*args, db_path=None)`
A local helper (not exported) that invokes `reasons.cli.main()` with a fabricated `sys.argv`, captures `stdout`/`stderr` via `StringIO` patches, and returns `(stdout_text, stderr_text, exit_code)`. It treats `SystemExit` as the normal termination path and maps `None` codes to `0`.

### `db` fixture
Builds a minimal belief network in a temporary SQLite database:

```
normal-premise-a  (IN)
normal-premise-b  (IN)
blocker-x         (IN)  ← active blocker
support-for-gated (IN)
gated-conclusion  (gated by blocker-x, supported by support-for-gated)
retracted-defect  (retracted, reason="Fixed in PR #99")
```

This gives the tests exactly 1 blocker, 1 gated belief, and 1 retracted premise — the minimum non-trivial scenario.

### `TestBuildStructuredReport`
Tests `_build_structured_report(in_count, out_count, gated_data, retracted, details)`. The helper `_make_gated_data` constructs the expected intermediate dict so each test isolates one formatting concern:

- **Title** is always present (`# Gated Beliefs Report`).
- **Summary stats** line reads `{in} IN / {out} OUT`.
- **Retracted premises** section appears only when the list is non-empty.
- **Active blockers** include the blocker's text, gated child IDs, and "Gates N belief(s)".
- **Empty network** produces a "No blockers or retracted premises found." sentinel.
- **Sort order**: blockers with more gated beliefs appear *first* (descending by `len(gated)`).

### `TestFormatDataForPrompt`
Verifies `_format_data_for_prompt` produces a plain-text summary suitable for inclusion in an LLM system/user prompt. Checks that counts, retracted entries, and blocker details are all present.

### `TestReportGatedApi`
Calls `api.report_gated(db_path=...)` and asserts:
- Returns a dict with keys `report`, `blocker_count`, `gated_count`, `retracted_count`.
- `report` is a `str` containing the expected IDs and retraction reason.
- An empty DB yields all-zero counts but still a valid report header.

### `TestReportGatedLlm`
Patches `reasons.llm.invoke_model` to verify:
- When `model=` is passed, `invoke_model` is called exactly once and its return value is embedded in the report.
- When no `model` is passed, `invoke_model` is **never** called.
- The LLM path still prepends the structured header and stats.
- `api.report_gated` forwards the `model` parameter correctly.

### `TestReportGatedCli`
End-to-end CLI tests:
- `--help` exits 0 and mentions the command.
- Default run writes the report to **stdout** and a summary line to **stderr**.
- `-o <file>` writes the report to the file, confirms existence, and prints a "Report written to" confirmation to stdout.

## Patterns

1. **Layered testing pyramid** — pure functions → orchestration → API → CLI. Each layer mocks or avoids the layers above it, so a regression in `_build_structured_report` doesn't require a live LLM call.
2. **Mock-at-the-boundary** — `reasons.llm.invoke_model` is the sole external I/O dependency; it's patched at the module level (`patch("reasons.llm.invoke_model", ...)`), not at the call site.
3. **CLI capture via `sys` patching** — the `run_cli` helper patches `sys.argv`, `sys.stdout`, `sys.stderr` simultaneously in a single `with` block, ensuring no cross-test contamination.
4. **Fixture-scoped temp DB** — `tmp_path` (pytest built-in) gives each test an isolated SQLite file; no shared state between tests.
5. **Helper-method data factories** — `_make_gated_data` and the inline dicts in `TestFormatDataForPrompt` keep the "arrange" step declarative and readable.

## Dependencies

**Imports (this file → other modules):**

| Import | Role |
|--------|------|
| `reasons.api` | Public entry points: `add_node`, `retract_node`, `report_gated` |
| `reasons.report_gated` | Under-test: `_build_structured_report`, `_format_data_for_prompt`, `report_gated` |
| `reasons.cli` | `main()` for CLI-level tests |
| `reasons.llm` | `invoke_model` (mocked, never called for real) |
| `pytest`, `unittest.mock` | Test framework & mocking |

**Imported by (what depends on this file):** Nothing imports test files in production. The file is consumed only by the test runner (`pytest`).

## Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  Test execution (per test class)                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Fixture `db` runs:                                          │
│       api.add_node(...) × 6                                     │
│       api.retract_node(...)                                     │
│       → returns db_path (SQLite)                                │
│                                                                 │
│  2. Test body:                                                  │
│     ┌──────────────────────────────────────────────┐            │
│     │ Pure-function tests (Test*)                  │            │
│     │   call _build_structured_report /            │            │
│     │   _format_data_for_prompt with literal dicts │            │
│     │   → assert on returned string                │            │
│     └──────────────────────────────────────────────┘            │
│     ┌──────────────────────────────────────────────┐            │
│     │ API tests (TestReportGatedApi)               │            │
│     │   api.report_gated(db_path=db)               │            │
│     │     → report_gated(...)                      │            │
│     │         → _build_structured_report(...)      │            │
│     │         → returns dict{report, counts}       │            │
│     └──────────────────────────────────────────────┘            │
│     ┌──────────────────────────────────────────────┐            │
│     │ LLM tests (TestReportGatedLlm)               │            │
│     │   patch(reasons.llm.invoke_model)            │            │
│     │   report_gated(..., model="claude")          │            │
│     │     → _format_data_for_prompt(...)           │            │
│     │     → llm.invoke_model(prompt) → narrative   │            │
│     │     → header + narrative → return str        │            │
│     └──────────────────────────────────────────────┘            │
│     ┌──────────────────────────────────────────────┐            │
│     │ CLI tests (TestReportGatedCli)               │            │
│     │   run_cli("report-gated", ...)               │            │
│     │     → sys.argv patched                       │            │
│     │     → cli.main() parses args                 │            │
│     │     → calls api.report_gated                 │            │
│     │     → prints to stdout / writes file         │            │
│     └──────────────────────────────────────────────┘            │
│                                                                 │
│  3. Assertions on captured output / return values               │
└─────────────────────────────────────────────────────────────────┘
```

## Invariants

- **Blocker ordering**: In both the structured report and the prompt, blockers are always sorted **descending** by the number of beliefs they gate. `test_blockers_sorted_by_gated_count` enforces this by checking `pos_many < pos_few`.
- **Report header is unconditional**: Regardless of network size, the first line is `# Gated Beliefs Report`.
- **LLM is opt-in**: `invoke_model` must not be called unless an explicit `model` string is supplied. The no-model path must still produce a valid structured report.
- **Return shape**: `api.report_gated` always returns a dict with exactly the keys `report` (str), `blocker_count` (int), `gated_count` (int), `retracted_count` (int).
- **Empty-state message**: When there are zero blockers *and* zero retracted premises, the report must contain the sentinel "No blockers or retracted premises found."
- **Retracted section is conditional**: The "Fixed Defects" section must be *absent* when the retracted list is empty (`test_no_retracted_premises`).

## Error Handling

This test file doesn't test error paths per se (no malformed inputs, missing DB, etc.), but it does reveal how errors are *conveyed*:

- **CLI exit codes**: `run_cli` catches `SystemExit` and normalises `e.code` (treating `None` as `0`). Tests assert `code == 0` for success paths. A non-zero code would indicate a CLI-level failure (bad flag, missing DB file, etc.).
- **No exceptions from pure functions**: `_build_structured_report` and `_format_data_for_prompt` are expected to return strings unconditionally; the tests never wrap them in `pytest.raises`.
- **LLM failure is not tested here**: Since `invoke_model` is always mocked to return a string, the test suite doesn't cover what happens when the LLM raises or times out. That concern likely lives in `reasons/llm.py` tests or is handled by a retry wrapper.
- **File I/O**: `test_file_output` opens the written file with plain `open().read()` — if the file doesn't exist, the test fails with `FileNotFoundError`. There's no explicit error-path test for an unwritable path.

---

## Topics to Explore

- [file] `reasons/report_gated.py` — The implementation under test; understanding `_build_structured_report`, `_format_data_for_prompt`, and `report_gated` side-by-side with these tests will reveal the full contract.
- [file] `reasons/report_belief.py` — The sibling "non-gated" belief report; comparing the two clarifies why the gated variant needs a separate report path (blockers, gating relationships, retracted premises).
- [function] `reasons/api.py:report_gated` — The public API wrapper that assembles counts from the storage layer and delegates to the report builder; useful for understanding where DB queries happen vs. where formatting happens.
- [file] `tests/test_report_belief.py` — The analogous test suite for the plain belief report; diffing the two test files highlights what the "gated" variant adds (blocker sorting, gated-child listing, retracted-premise section).
- [general] `llm-invocation-contract` — How `reasons.llm.invoke_model` is called (prompt structure, model parameter, return type) and whether there's a shared prompt-template convention across `report_gated`, `report_belief`, and `ask`.

---

## Beliefs

- `report-gated-blockers-sorted-descending-by-gated-count` — In both the structured Markdown report and the LLM prompt, blockers are always ordered with the most gated beliefs first; this is asserted by `test_blockers_sorted_by_gated_count` and is not a caller choice.
- `report-gated-llm-call-is-opt-in` — `invoke_model` in `reasons.llm` is called if and only if the `model` parameter is a non-None string; omitting it must not produce an LLM call nor an error, and the structured report is returned as-is.
- `api-report-gated-returns-fixed-dict-shape` — `api.report_gated` always returns a dict with exactly the keys `report` (str), `blocker_count` (int), `gated_count` (int), and `retracted_count` (int), even when all counts are zero.
- `retracted-section-conditional-on-nonempty-list` — The "Fixed Defects" section in the structured report is present only when the retracted-premises list is non-empty; an empty list must produce no such section (asserted by `test_no_retracted_premises`).
- `cli-report-gated-stdout-vs-file-mutual-exclusion` — Without `-o`, the report body goes to stdout and a summary to stderr; with `-o <path>`, the body goes to the file and stdout carries only a "Report written to" confirmation. The two output modes are mutually exclusive per invocation.
