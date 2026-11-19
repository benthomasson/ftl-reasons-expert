---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_review.py

## Purpose

`tests/test_review.py` is the behavioral contract test suite for the derived-belief review pipeline. It covers three layers of the project:

1. **Prompt formatting and LLM response parsing** — `format_belief_for_review()` and `parse_review_response()`.
2. **Batch review orchestration and filtering** — `reasons.review.review_beliefs()` and `api.review_beliefs()`.
3. **CLI and persistence behavior** — the `review-beliefs` command, auto-retraction, markdown findings, JSON reports, metadata writes, and `list_nodes()` review filters.

The tests are mostly black-box: they construct either an in-memory-style exported `nodes` dict or a temporary SQLite database, mock the LLM subprocess, invoke the public function/CLI/API, and assert on return values, stdout, files, and database state.

## Key Components

### Helpers

- `run_cli(*args, db_path=None)`
  - Builds argv as `["reasons", "--db", db_path, *args]` when `db_path` is supplied.
  - Patches `sys.argv`, `sys.stdout`, and `sys.stderr`.
  - Calls `reasons.cli.main()`.
  - Catches `SystemExit` and returns `(stdout, stderr, exit_code)`.
  - If `main()` returns normally, returns exit code `0`.

- `_make_nodes()`
  - Returns a minimal exported-node dictionary matching the project’s node schema:
    - `text`
    - `truth_value` (`"IN"` / `"OUT"`)
    - `justifications` list with `type`, `antecedents`, `outlist`, and `label`
  - Contains:
    - three premises: `premise-a`, `premise-b`, `premise-c`
    - `derived-ab`: an `IN` belief justified by `premise-a` and `premise-b`
    - `gated-abc`: an `OUT` belief with an `outlist` containing `premise-c`
    - `derived-deep`: an `IN` belief depending on `derived-ab` and `premise-c`

### Test Classes

- `TestFormatBeliefForReview`
  - Defines the expected markdown-like review prompt format.
  - Verifies:
    - belief heading `### <id>`
    - belief text
    - antecedent lines like `premise-a: A is true`
    - outlist section `Unless (must be OUT):`
    - labels `Label: ...`
    - multiple justifications numbered as `Justification 1/2:`
    - single justification is not numbered
    - no-justification beliefs omit `Antecedents:`
    - missing antecedents render as `nonexistent: (not found in network)`
    - missing belief ID returns `""`

- `TestParseReviewResponse`
  - Defines the LLM response parser contract.
  - Verifies:
    - valid JSON list parsing
    - extraction of a JSON list from surrounding prose
    - ignoring prose brackets before the actual JSON array
    - malformed JSON returns `[]`
    - non-list JSON returns `[]`
    - items missing `id` are skipped
    - missing fields receive defaults:
      - `valid=True`
      - `sufficient=True`
      - `necessary=True`
      - `unnecessary_antecedents=[]`
      - `comment=""`
      - `defeat_reason_type=""`
    - optional fields such as `defeat_reason_type` are preserved

- `TestReviewBeliefs`
  - Tests the lower-level `review_beliefs(nodes, ...)` function.
  - Contracts:
    - only reviews derived `IN` beliefs with justifications
    - filters requested `belief_ids` to existing/reviewable nodes
    - respects `batch_size`
    - passes `timeout` through to the LLM subprocess
    - returns parsed review results
    - returns `[]` when there are no reviewable derived beliefs
    - swallows per-batch LLM failures and continues

- `TestReviewBeliefsApi`
  - Tests `api.review_beliefs()` against a real temporary SQLite DB.
  - Verifies filters:
    - derived-only
    - `min_depth`
    - `sample`
    - `depends_on`
    - `namespace` (`"eng"` includes `eng:...`; empty namespace means local IDs without `:`)
    - `visible_to` / access tags
  - Verifies summary counts:
    - `reviewed`
    - `total_derived`
    - `invalid`
    - `insufficient`
    - `unnecessary`
  - Verifies `results` contains parsed review entries.

- `TestCmdReviewBeliefs`
  - Tests the `review-beliefs` CLI command.
  - Verifies:
    - `--auto-retract` retracts invalid beliefs by setting truth value to `OUT`
    - `--dry-run` prevents retraction
    - `-o` writes markdown findings with header `# Belief Review Findings`
    - CLI prints flags like `INVALID`, `INSUFFICIENT`, `UNNECESSARY(premise-a)`
    - no derived beliefs prints `No derived beliefs to review.`
    - `--report-dir` writes a JSON report containing `timestamp`, `status="complete"`, `reviewed`, `total_derived`, and `results`
    - `--no-report` suppresses report creation
    - `on_batch` callback fires with partial results after each batch

- `TestReviewBeliefsMetadata`
  - Verifies persisted review metadata after `api.review_beliefs()`:
    - `reviewed_at` is written
    - `metadata["review_result"]` is classified
    - classification precedence: invalid beats insufficient, etc.
    - `dry_run=True` skips metadata writes

- `TestListNodesReviewFilters`
  - Tests `api.list_nodes()` review-related filters:
    - `never_reviewed=True` returns unreviewed derived beliefs but not premises
    - `not_reviewed_since=30` includes never-reviewed beliefs and excludes recently reviewed ones
    - `by_impact=True` sorts by `dependent_count` descending
    - returned nodes include `last_reviewed` and `review_result`

## Patterns

- **Pytest class grouping by concern**: formatting, parsing, lower-level review, API review, CLI, metadata, and list filters are separated into focused test classes.
- **Temporary real database fixtures**: API/CLI/metadata tests use `tmp_path` and `api.add_node()` to create a real SQLite database instead of mocking storage.
- **Subprocess-level LLM mocking**: tests patch:
  - `reasons.llm.shutil.which`
  - `reasons.llm.subprocess.run`

  This prevents actual Claude CLI invocation and lets tests inject canned JSON responses or raise errors.
- **Dynamic mock result objects**: tests create a lightweight object with `returncode`, `stdout`, and `stderr` to mimic `subprocess.CompletedProcess`.
- **Behavioral assertions rather than snapshot assertions**: formatting tests use substring checks such as `"### derived-ab"` and `"Label: combined observation"`, so output can evolve as long as the documented pieces remain.
- **Hermetic CLI testing**: `run_cli()` patches `sys.argv`, stdout, and stderr, and catches `SystemExit`, making CLI tests safe to run in-process.

## Dependencies

### Imports from production code

- `reasons.review.format_belief_for_review`
- `reasons.review.parse_review_response`
- `reasons.review.review_beliefs`
- `reasons.review.REVIEW_BATCH_SIZE`
- `reasons.api`
- `reasons.cli.main`

### Indirect dependency

- `reasons.llm`
  - The tests do not import it directly, but they patch `reasons.llm.shutil.which` and `reasons.llm.subprocess.run`. This means `review_beliefs()` must process LLM work through `reasons.llm` internals, not by calling `subprocess` directly from `reasons.review`.

### External dependencies

- `json`
- `sys`
- `io.StringIO`
- `unittest.mock.patch`
- `pytest`
- `os` locally inside a few CLI/report tests

### Imported by

- Pytest test discovery only. No production module should import this test file.

## Flow

A typical test follows this sequence:

1. Build input:
   - lower-level tests call `_make_nodes()`
   - API/CLI tests create a temp DB via `api.add_node()`
2. Build a mocked LLM JSON response.
3. Patch LLM subprocess lookup and execution.
4. Invoke the function, API, or CLI.
5. Assert on:
   - returned review results
   - summary counts
   - database truth values
   - metadata fields
   - stdout contents
   - generated markdown/JSON files

For CLI tests, `run_cli()` sets argv with the database option before the subcommand, invokes `main()`, captures output, and returns the exit code. The tests then inspect either stdout, the SQLite DB, or files written by `-o` / `--report-dir`.

For API tests, `api.review_beliefs()` is responsible for selecting reviewable beliefs, applying filters such as depth/namespace/sample/access tags, invoking lower-level review logic, and persisting metadata unless `dry_run` is set.

## Invariants

- A reviewable belief is a derived `IN` node with at least one justification. Premises and `OUT` gated beliefs are not reviewed.
- `format_belief_for_review("missing-id", nodes)` returns `""`.
- Missing antecedents are rendered gracefully as `"(not found in network)"`.
- Multiple justifications are numbered only when there is more than one.
- `parse_review_response()` only accepts top-level JSON arrays. It ignores non-list JSON and malformed JSON.
- Review response entries without an `id` are dropped.
- Missing response fields are normalized with defaults.
- `review_beliefs()` honors an explicit `batch_size`; with two derived `IN` beliefs and `batch_size=1`, it makes two LLM calls.
- LLM subprocess failures do not propagate out of `review_beliefs()`; they result in skipped/empty results.
- `api.review_beliefs()` supports filters: `min_depth`, `sample`, `depends_on`, `namespace`, and `visible_to`.
- `namespace=""` means local/unqualified IDs only; namespaced IDs contain `:`.
- `dry_run=True` prevents metadata writes.
- CLI `--auto-retract` sets invalid beliefs’ truth value to `OUT`; `--dry-run` prevents that mutation.
- `--no-report` prevents creation of the report directory.
- `list_nodes(never_reviewed=True)` returns unreviewed derived beliefs, not premises.
- `list_nodes(by_impact=True)` sorts by `dependent_count` descending.
- `list_nodes()` results include review-related fields such as `last_reviewed` and `review_result`.

## Error Handling

- `parse_review_response()` is deliberately tolerant:
  - malformed JSON returns `[]`
  - non-list JSON returns `[]`
  - items without `id` are skipped
  - missing fields are defaulted
- `format_belief_for_review()` handles missing antecedent IDs without raising.
- `review_beliefs()` catches LLM/subprocess failures per batch; the test `test_llm_error_continues` expects `[]` rather than an exception when the only batch fails.
- The CLI helper catches `SystemExit` from `main()` so tests can inspect exit codes without terminating pytest.
- No test expects production code to raise for malformed LLM output; the parser’s fallback behavior is the contract.

## Topics to Explore

- [file] `reasons/review.py` — Implements the prompt formatter, response parser, batching, and lower-level review orchestration tested here.
- [function] `reasons/review.py:review_beliefs` — The core batching function whose filtering, timeout, and error behavior this suite pins down.
- [function] `reasons/api.py:review_beliefs` — Applies depth/sample/dependency/namespace/access filters and persists review metadata.
- [file] `reasons/cli.py` — Contains the `review-beliefs` command, auto-retract behavior, markdown output, and JSON report writing.
- [general] `llm-mocking-contract` — The tests assume review LLM calls go through `reasons.llm.shutil.which` and `reasons.llm.subprocess.run`, which is important when refactoring the LLM boundary.

## Beliefs

- `review-tests-mock-llm-subprocess` — Lower-level review tests patch `reasons.llm.shutil.which` and `reasons.llm.subprocess.run`, so they never invoke a real Claude process.
- `review-beliefs-selects-in-derived-nodes-with-justifications` — The review set is derived `IN` nodes with non-empty justifications; premises and `OUT` gated beliefs are excluded.
- `parse-review-response-returns-empty-on-malformed-json` — Malformed JSON, non-list JSON, and items missing `id` do not produce review entries.
- `api-review-beliefs-persists-review-metadata` — Successful API review calls write `reviewed_at` and `metadata.review_result` unless `dry_run=True`.
- `cli-auto-retract-sets-invalid-derived-nodes-to-out` — `review-beliefs --auto-retract` changes invalid reviewed beliefs to `truth_value="OUT"`, while `--dry-run` prevents the mutation.
