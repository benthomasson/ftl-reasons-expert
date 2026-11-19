---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_review_justifications.py

# `tests/test_review_justifications.py`

## Purpose

This file is a pytest suite for the “review justifications” workflow. It validates the LLM-assisted classification of multi-antecedent support-link justifications as:

- `ANY` — antecedents independently support the conclusion
- `ALL` — all antecedents are required together
- `MIXED` — some required, some independent

The tests exercise two layers:

1. `reasons.review_justifications` — parsing LLM output, selecting candidate nodes, batching/parallel invocation.
2. `reasons.api.review_justifications` — loading a real temp DB, running the review, producing summary counts, and confirming the DB is not modified.

The suite targets the `reasons` package, not the mirrored `reasons_lib` package. LLM calls are always mocked by patching `reasons.review_justifications.invoke_model`.

## Key Components

### Fixture: `network_db`

Creates a temporary SQLite DB with `tempfile.mkdtemp()` and populates it via `api.add_node`:

- `fact-a`, `fact-b`, `fact-c` — premise-like facts
- `derived-1` — SL justification with `["fact-a", "fact-b"]`
- `derived-2` — SL justification with `["fact-a", "fact-b", "fact-c"]`
- `single-ant` — SL justification with `["fact-a"]`

This fixture lets API tests verify real storage/serialization rather than only in-memory dicts. It does not explicitly clean up the temp directory.

### `TestParseJustificationReview`

Tests `parse_justification_review(response)`.

Observed contract:

- Extracts a JSON array even when surrounded by prose.
- Returns a list of dicts containing `id`, `classification`, `required_antecedents`, `independent_antecedents`, and `comment`.
- Skips array items without an `id`.
- Defaults missing or `None` classification to `"ALL"`.
- Defaults missing or `None` list fields to `[]`.
- Defaults missing or `None` comment to `""`.
- Normalizes classification case, e.g. `"any"` → `"ANY"`.
- Returns `[]` when no JSON is found.

### `TestHasMultiAntecedentSl`

Tests `_has_multi_antecedent_sl(node, min_antecedents)`.

Observed contract:

- Looks at `node["justifications"]`.
- Returns `True` if any justification has `type == "SL"` and `len(antecedents) >= min_antecedents`.
- Returns `False` for missing `justifications`, empty justifications, or too few antecedents.
- Threshold is inclusive: with `min_antecedents=3`, two antecedents are false and three are true.

### `TestReviewJustifications`

Tests the core `review_justifications(nodes, batch_size=..., parallel=..., min_antecedents=...)` function.

Covered behavior:

- Filters out nodes without a qualifying multi-antecedent SL justification.
- Excludes single-antecedent nodes and premise-like nodes with no justifications.
- Calls `invoke_model` to classify selected nodes.
- Combines parsed results into a single result list.
- Supports `batch_size` and `parallel`; with `batch_size=1, parallel=2`, the mocked LLM can respond separately per candidate and both results are preserved.
- Honors `min_antecedents`: with `min_antecedents=3`, only the three-antecedent node is reviewed.

The parallel test’s mock signature accepts `(prompt, model=None, timeout=None)`, indicating the production call may pass model/timeout keyword arguments.

### `TestApiReviewJustifications`

Tests `api.review_justifications(db_path=..., belief_ids=...)`.

Observed behavior:

- Loads nodes from the DB, runs review, and returns a summary dict.
- Summary keys asserted:
  - `reviewed` — total results
  - `convert_any` — count of `"ANY"`
  - `keep_all` — count of `"ALL"`
  - `convert_mixed` — count of `"MIXED"`
- `belief_ids=["derived-1"]` restricts review to that node.
- The review is read-only: after calling `api.review_justifications`, `derived-1` still has exactly one justification with two antecedents.

## Patterns

- **Class-based pytest organization**: parser, helper, core orchestration, and API integration concerns are separated.
- **Mocked LLM boundary**: `reasons.review_justifications.invoke_model` is patched directly, keeping tests deterministic and offline.
- **Side-effect mocking for batching/concurrency**: `test_parallel` uses `side_effect` to return different JSON depending on which node text appears in the prompt.
- **Real storage integration for API tests**: the fixture uses `api.add_node` against a temp DB, so DB loading and node lookup are exercised.
- **Tolerant LLM-output contract**: parser tests encode robustness to prose, missing fields, null fields, and missing IDs.
- **Advisory/read-only command semantics**: the API test explicitly verifies no DB mutation.

One maintenance note: `MagicMock` is imported but not used in this file.

## Dependencies

### This file imports

- `json`, `os`, `tempfile` — JSON fixtures and temp DB path creation.
- `pytest` — fixtures and test discovery.
- `unittest.mock.patch` — replace `invoke_model`.
- `reasons.api` — node creation, node lookup, and API-level review entrypoint.
- `reasons.review_justifications`:
  - `parse_justification_review`
  - `review_justifications`
  - `_has_multi_antecedent_sl`

### Production code under test

- `reasons/review_justifications.py` — parser, candidate filtering, LLM invocation, batching.
- `reasons/api.py:review_justifications` — DB-backed wrapper and summary generation.

### What depends on this test file

Only the test runner/CI. No production module imports this file.

## Flow

1. For API tests, pytest invokes `network_db`, which creates a temp DB and inserts fact and derived nodes using `api.add_node`.
2. Parser tests pass synthetic LLM response strings into `parse_justification_review`; the parser extracts and normalizes the JSON array.
3. Helper tests construct small node dicts and call `_has_multi_antecedent_sl`.
4. Core review tests build in-memory `nodes` dicts, patch `invoke_model`, and call `review_justifications`.
   - Candidate nodes are filtered by multi-antecedent SL presence and `min_antecedents`.
   - Mocked LLM responses are parsed and combined.
5. API tests patch `invoke_model`, call `api.review_justifications`, and assert summary counts derived from classifications.
6. The final API test reads the node back with `api.show_node` and confirms the review did not write changes.

## Invariants

- A node is only a review candidate if it has at least one `SL` justification whose antecedent list length meets `min_antecedents`.
- Single-antecedent SL nodes are excluded from the default tested review path.
- Missing/null classification defaults to `"ALL"`.
- Classification values are normalized to uppercase.
- Items without an `id` are discarded.
- `required_antecedents` and `independent_antecedents` are always lists in parsed output, defaulting to `[]`.
- The review command is read-only with respect to the belief network DB.
- Summary counters correspond directly to parsed classifications: `convert_any`, `keep_all`, and `convert_mixed`.

## Error Handling

- `parse_justification_review` is intentionally forgiving: if no JSON can be found, it returns `[]` rather than raising.
- Malformed individual array items without an `id` are skipped.
- `None` fields are normalized rather than causing attribute/type errors.
- `_has_multi_antecedent_sl` safely returns `False` when `justifications` is missing.
- These tests do not cover:
  - LLM invocation failures
  - invalid JSON that looks like JSON but is malformed
  - unsupported classification values
  - non-list antecedent fields
  - DB errors

So the current suite documents tolerant parsing and read-only success paths, but not broader failure propagation.

## Topics to Explore

- [file] `reasons/review_justifications.py` — The implementation under test, including JSON extraction, candidate filtering, batching, and concurrency.
- [function] `reasons/review_justifications.py:parse_justification_review` — Inspect how the parser locates JSON arrays inside prose and normalizes malformed/null fields.
- [function] `reasons/api.py:review_justifications` — See how nodes are loaded, how `belief_ids` restrict the review, and how summary counts are produced.
- [function] `reasons/llm.py:invoke_model` — The mocked LLM boundary; understanding its prompt/model/timeout contract explains the parallel test mock.
- [general] `llm-json-parsing-strategy` — Compare how other modules in `reasons/` extract and validate JSON from model responses.

## Beliefs

- `parse-defaults-classification-to-all` — `parse_justification_review` normalizes missing, null, or omitted classification to `"ALL"`.
- `candidate-filter-requires-multi-antecedent-sl` — `review_justifications` only reviews nodes containing an `SL` justification whose antecedent count meets or exceeds `min_antecedents`.
- `review-api-is-read-only` — `api.review_justifications` returns summary counts and the tests confirm the underlying justification/antecedent data is unchanged afterward.
- `parallel-batching-preserves-results` — With `batch_size=1` and `parallel=2`, `review_justifications` invokes `invoke_model` per candidate batch and combines both parsed responses.
- `api-summary-counts-classifications` — The API summary reports `reviewed` as result count and maps `ANY`, `ALL`, and `MIXED` classifications to `convert_any`, `keep_all`, and `convert_mixed`.
