# File: tests/test_review.py

**Date:** 2026-05-08
**Time:** 14:09

## Purpose

`tests/test_review.py` is the test suite for the **belief review pipeline** — the subsystem that uses an LLM to validate whether derived beliefs actually follow from their justifications. It tests three layers: the formatting/parsing utilities in `reasons_lib.review`, the API-level `api.review_beliefs()` orchestrator, and the `review-beliefs` CLI command. This file owns coverage for the entire review vertical, from prompt construction through retraction side-effects.

## Key Components

### Helpers

**`run_cli(*args, db_path=None)`** — Invokes the CLI's `main()` in-process by patching `sys.argv`, `sys.stdout`, and `sys.stderr`. Returns `(stdout, stderr, exit_code)`. This avoids subprocess overhead and lets tests inspect output directly.

**`_make_nodes()`** — Builds a hand-crafted node graph with known structure: three premises (`premise-a`, `premise-b`, `premise-c`), one straightforward derived belief (`derived-ab`), one gated belief with an outlist (`gated-abc`), and one deeper derived belief (`derived-deep` depends on `derived-ab`). This graph exercises all the formatting edge cases: antecedents, outlists, missing nodes, multi-depth chains.

### Test Classes

| Class | Layer Tested | What It Proves |
|---|---|---|
| `TestFormatBeliefForReview` | `review.format_belief_for_review` | Prompt construction: antecedents, outlists, missing nodes, multi-justification numbering, premises with no justifications |
| `TestParseReviewResponse` | `review.parse_review_response` | LLM output extraction: valid JSON, JSON embedded in prose, malformed input, default field values, filtering items without IDs, non-list JSON rejection |
| `TestReviewBeliefs` | `review.review_beliefs` | Orchestration: batching, timeout passthrough, filtering to existing IDs, graceful LLM errors, empty-derived short-circuit |
| `TestReviewBeliefsApi` | `api.review_beliefs` | API layer: derived-only filtering, `min_depth`, `sample`, `depends_on`, `visible_to` (access tags), summary count aggregation |
| `TestCmdReviewBeliefs` | CLI `review-beliefs` | End-to-end: `--auto-retract` actually changes truth values, `--dry-run` prevents mutations, `-o` writes findings markdown, `--report-dir` writes JSON reports, `--no-report` suppresses them, `on_batch` callback |
| `TestReviewBeliefsMetadata` | `api.review_beliefs` metadata writes | Post-review metadata: `last_reviewed` timestamp, `review_result` classification (`pass`/`invalid`/`insufficient`), priority ordering (invalid > insufficient > pass), dry-run skips metadata |
| `TestListNodesReviewFilters` | `api.list_nodes` review-aware filters | Query filters: `never_reviewed`, `not_reviewed_since`, `by_impact` sorting, presence of `last_reviewed`/`review_result` fields |

## Patterns

**LLM isolation via double-patching.** Every test that would call an LLM patches both `reasons_lib.llm.shutil.which` (to make the LLM discovery succeed) and `reasons_lib.llm.subprocess.run` (to return canned JSON). This means the LLM module uses `shutil.which` to locate the `claude` binary and `subprocess.run` to invoke it — tests never touch a real LLM.

**Inline mock result objects.** Rather than importing a mock class, tests construct anonymous result objects with `type("R", (), {"returncode": 0, "stdout": ..., "stderr": ""})()`. This is compact but means the mock shape must be kept in sync with what `subprocess.run` actually returns.

**Fixture-based DB setup.** The API and CLI test classes use `@pytest.fixture` methods named `db_path` that create a temporary SQLite database via `api.add_node()` calls. Each fixture builds a specific graph topology tailored to the tests in that class.

**Three-layer testing.** The file mirrors the production architecture: pure functions (`format_belief_for_review`, `parse_review_response`), an API orchestrator (`api.review_beliefs`), and a CLI entry point (`run_cli` → `main()`). Each layer is tested independently with its own mocking boundary.

## Dependencies

**Imports from production code:**
- `reasons_lib.review`: `format_belief_for_review`, `parse_review_response`, `review_beliefs`, `REVIEW_BATCH_SIZE` (the constant is imported but not directly asserted — batch behavior is tested via call counts)
- `reasons_lib.api`: the public API for node CRUD and `review_beliefs`
- `reasons_lib.cli.main`: the CLI entry point

**Transitive dependencies patched:** `reasons_lib.llm.shutil.which` and `reasons_lib.llm.subprocess.run` — the review pipeline delegates LLM calls through `reasons_lib.llm`.

**Nothing imports this file** — it's a test module.

## Flow

A typical review cycle tested here:

1. **Build nodes** — either via `_make_nodes()` (dict) or `api.add_node()` (DB)
2. **Filter** — select derived beliefs by ID, depth, dependency, access tags, or sample count
3. **Format** — `format_belief_for_review` renders each belief + its justification graph into a markdown prompt
4. **Send to LLM** — `review_beliefs` batches beliefs (controlled by `batch_size`), calls the LLM subprocess
5. **Parse response** — `parse_review_response` extracts a JSON array from possibly-prosaic LLM output
6. **Classify** — each result is scored on three axes: valid, sufficient, necessary
7. **Act** — optionally retract invalid beliefs (`--auto-retract`), write metadata (`last_reviewed`, `review_result`), emit findings markdown (`-o`), write JSON reports (`--report-dir`)

## Invariants

- **Only derived beliefs are reviewed.** Premises (nodes with empty `justifications`) are always excluded. `test_filters_to_derived_only` and `test_empty_derived_returns_empty` enforce this.
- **Missing node IDs produce graceful degradation, not crashes.** `test_missing_antecedent_graceful` shows `(not found in network)` placeholders; `test_missing_node_returns_empty` returns `""`.
- **`parse_review_response` never raises.** Malformed JSON → `[]`. Non-list JSON → `[]`. Items without `id` → skipped. Missing fields → defaults (`valid=True`, `sufficient=True`, `necessary=True`).
- **Review result classification priority: `invalid` > `insufficient` > `pass`.** When all three axes fail, the result is `"invalid"`, not `"insufficient"`. `test_review_result_classification_priority` enforces this.
- **`--dry-run` is side-effect-free.** It prevents both retraction (`test_dry_run_prevents_retraction`) and metadata writes (`test_dry_run_skips_metadata`).
- **`--no-report` suppresses report directory creation entirely** — the directory shouldn't even exist afterward.

## Error Handling

- **LLM failures are swallowed.** `test_llm_error_continues` shows that a `RuntimeError` from `subprocess.run` produces an empty results list, not an exception. The review pipeline is designed to degrade gracefully rather than abort.
- **Parse failures are silent.** `parse_review_response` returns `[]` for any unparseable input — no warnings, no exceptions.
- **CLI captures `SystemExit`.** `run_cli` wraps `main()` in a try/except for `SystemExit`, capturing the exit code. Non-zero codes can be asserted but aren't heavily tested here.

## Topics to Explore

- [file] `reasons_lib/review.py` — The production code under test; see how `format_belief_for_review` constructs the LLM prompt and how batching is implemented
- [function] `reasons_lib/api.py:review_beliefs` — The API orchestrator that ties together filtering, LLM calls, metadata writes, and summary aggregation
- [file] `reasons_lib/llm.py` — The LLM subprocess interface that all tests mock; understanding its contract explains why `shutil.which` and `subprocess.run` are the two patch targets
- [general] `review-result-classification` — How `invalid`/`insufficient`/`pass` are prioritized and how that maps to `--auto-retract` behavior
- [file] `tests/test_check_stale.py` — Complementary test file covering staleness detection, which intersects with `not_reviewed_since` filtering tested here

## Beliefs

- `review-only-validates-derived-beliefs` — The review pipeline filters out premises (nodes with no justifications) before sending anything to the LLM; premises are never reviewed
- `parse-review-response-never-raises` — `parse_review_response` returns an empty list on any malformed input (bad JSON, non-list JSON, missing IDs) rather than raising exceptions
- `review-result-priority-invalid-over-insufficient` — When classifying review results, `invalid` takes priority over `insufficient`; a belief that fails all three axes is classified as `"invalid"`, not `"insufficient"`
- `dry-run-prevents-both-retraction-and-metadata` — The `dry_run` flag prevents both truth-value changes (retraction) and metadata side-effects (`last_reviewed`, `review_result`), making it fully read-only
- `llm-subprocess-mock-requires-two-patches` — Tests must patch both `reasons_lib.llm.shutil.which` (binary discovery) and `reasons_lib.llm.subprocess.run` (invocation) to fully isolate from real LLM calls

