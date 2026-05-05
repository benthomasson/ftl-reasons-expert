# File: tests/test_review.py

**Date:** 2026-05-05
**Time:** 15:33

No problem — I have enough from the test file itself to give a thorough explanation.

---

## Purpose

`tests/test_review.py` is the test suite for the **derived belief validation** pipeline. It tests the ability to send derived beliefs (those with SL justifications) to an LLM for logical review, parse the LLM's structured response, and act on findings — including auto-retracting invalid beliefs and writing findings reports. It covers three layers: the pure formatting/parsing functions in `reasons_lib.review`, the `api.review_beliefs` orchestration function, and the `reasons review-beliefs` CLI command.

## Key Components

### Helper functions

- **`run_cli(*args, db_path=None)`** — Invokes the `reasons` CLI in-process by patching `sys.argv`, `sys.stdout`, and `sys.stderr`. Returns `(stdout, stderr, exit_code)`. This avoids subprocess overhead and lets tests inspect CLI output as strings. Used across several test files in this project (same pattern appears in `test_cli.py` etc.).

- **`_make_nodes()`** — Factory that builds a minimal exported-nodes dictionary with 6 nodes: three premises (`premise-a`, `premise-b`, `premise-c`), two derived beliefs (`derived-ab`, `derived-deep`), and one gated belief (`gated-abc` with an outlist). This is the canonical test fixture for the unit-level review functions that operate on raw node dicts rather than the database.

### Test classes

| Class | Layer tested | What it validates |
|---|---|---|
| `TestFormatBeliefForReview` | `review.format_belief_for_review` | Markdown rendering of a belief card: antecedents, outlist, labels, missing nodes, multiple justifications, numbering rules |
| `TestParseReviewResponse` | `review.parse_review_response` | JSON extraction from LLM output: valid JSON, embedded JSON in prose, malformed input, default field values, items without IDs, non-list JSON |
| `TestReviewBeliefs` | `review.review_beliefs` | End-to-end batch review: batching, filtering to existing IDs, batch size, timeout passthrough, LLM error resilience |
| `TestReviewBeliefsApi` | `api.review_beliefs` | API-level orchestration: filtering to derived-only, `min_depth`, `sample`, `depends_on`, `visible_to` (access tags), summary counts |
| `TestCmdReviewBeliefs` | CLI `review-beliefs` command | CLI integration: `--auto-retract`, `--dry-run`, `-o` output file, flag display, empty-network message |

## Patterns

### LLM mocking strategy

Every test that would invoke an LLM patches two things:
1. `reasons_lib.llm.shutil.which` → returns a fake path so the LLM backend selection doesn't fail
2. `reasons_lib.llm.subprocess.run` → returns a crafted `mock_result` object with `.returncode`, `.stdout`, `.stderr`

The mock result is built inline via `type("R", (), {...})()` — a one-liner anonymous class. This avoids importing `MagicMock` for a simple data carrier and keeps the test self-contained.

### Fixture layering

The unit tests (`TestFormatBeliefForReview`, `TestParseReviewResponse`) use `_make_nodes()` — a plain dict, no database. The integration tests (`TestReviewBeliefsApi`, `TestCmdReviewBeliefs`) use a `db_path` pytest fixture that builds a real SQLite database via `api.add_node`, exercising the full storage stack.

### CLI testing without subprocess

`run_cli` captures stdout/stderr via `StringIO` patches and catches `SystemExit`, which is the standard argparse exit mechanism. This lets tests assert on CLI output strings while staying in-process.

## Dependencies

**Imports:**
- `reasons_lib.review` — the module under test: `format_belief_for_review`, `parse_review_response`, `review_beliefs`, `REVIEW_BATCH_SIZE`
- `reasons_lib.api` — higher-level orchestration; used to set up DB fixtures and test `api.review_beliefs`
- `reasons_lib.cli.main` — entry point for CLI integration tests
- `reasons_lib.llm` — not imported directly but patched at `reasons_lib.llm.subprocess.run` and `reasons_lib.llm.shutil.which`

**Imported by:** Nothing — this is a test module.

## Flow

The review pipeline works in three stages, each tested by its own class:

1. **Format** (`format_belief_for_review`): Takes a belief ID and the full nodes dict. Produces a Markdown card listing the belief text, each justification's antecedents (resolved to their text), outlist entries, and labels. Multiple justifications get numbered headers; single justifications don't.

2. **Send & Parse** (`review_beliefs` → `parse_review_response`): Batches derived beliefs into groups of `REVIEW_BATCH_SIZE`, sends each batch to an LLM, and parses the response. The parser looks for a JSON array in the LLM output (tolerating surrounding prose), extracts it, applies defaults for missing fields (`valid=True`, `sufficient=True`, `necessary=True`, `unnecessary_antecedents=[]`, `comment=""`), and drops entries without an `id` field.

3. **Act** (CLI `review-beliefs`): Displays flags (`INVALID`, `INSUFFICIENT`, `UNNECESSARY(...)`) for each finding. With `--auto-retract`, retracts beliefs marked invalid. With `--dry-run`, skips retraction. With `-o`, writes a Markdown findings report.

## Invariants

- **Only derived beliefs are reviewed.** Premises (no justifications) are excluded. `test_empty_derived_returns_empty` and `test_filters_to_derived_only` verify this.
- **Missing fields default to "passing."** A review response of `[{"id": "x"}]` is treated as valid, sufficient, and necessary — the LLM only needs to report failures.
- **Items without `id` are dropped.** The parser silently skips response entries that lack an `id` field rather than crashing.
- **Non-list JSON is rejected.** A bare JSON object `{...}` is not treated as a single-element review — the parser requires a JSON array.
- **Batch size is respected.** When `batch_size=1` and there are 2 derived beliefs, exactly 2 LLM calls are made.
- **`--dry-run` prevents mutation.** The belief stays IN in the database even when the LLM says it's invalid.
- **LLM errors don't crash the pipeline.** `test_llm_error_continues` verifies that a `RuntimeError` from `subprocess.run` results in an empty list, not an exception.

## Error Handling

- **LLM failure:** `review_beliefs` catches exceptions from the LLM subprocess and returns an empty results list (`test_llm_error_continues`).
- **Malformed JSON:** `parse_review_response` returns `[]` when the response contains no parseable JSON array (`test_malformed_json_returns_empty`).
- **Missing antecedents:** `format_belief_for_review` renders `"(not found in network)"` for antecedent IDs that don't exist in the nodes dict, rather than crashing.
- **Missing belief:** `format_belief_for_review` returns an empty string for a nonexistent node ID.
- **CLI exit codes:** `run_cli` catches `SystemExit` and returns the exit code, so tests can assert on both output and exit status.

## Topics to Explore

- [file] `reasons_lib/review.py` — The implementation of `format_belief_for_review`, `parse_review_response`, and `review_beliefs` that this test suite exercises
- [function] `reasons_lib/api.py:review_beliefs` — The API-level orchestration that applies filtering (min_depth, sample, depends_on, visible_to) before calling the review module
- [file] `reasons_lib/llm.py` — The LLM invocation layer that `review_beliefs` delegates to; understanding how `subprocess.run` is called explains why the mocking targets `reasons_lib.llm.subprocess.run`
- [general] `outlist-gated-beliefs` — How GATE beliefs (outlist-based non-monotonic reasoning) interact with the review pipeline, and why `gated-abc` is OUT in the test fixture despite having IN antecedents
- [file] `tests/test_check_stale.py` — Another test suite that validates belief staleness, complementary to review's validity checking

## Beliefs

- `review-skips-premises` — `review_beliefs` only sends beliefs with at least one justification to the LLM; premise nodes (empty justifications list) are never reviewed
- `parse-review-defaults-to-passing` — `parse_review_response` defaults missing fields to valid=True, sufficient=True, necessary=True — the LLM only needs to report failures explicitly
- `review-tolerates-llm-failure` — When the LLM subprocess raises an exception, `review_beliefs` catches it and returns an empty list rather than propagating the error
- `review-parse-requires-json-array` — `parse_review_response` only accepts JSON arrays; a bare JSON object is treated as unparseable and returns an empty list
- `auto-retract-respects-dry-run` — The `--auto-retract` flag in `review-beliefs` CLI is gated by `--dry-run`, which prevents any database mutation while still displaying findings

