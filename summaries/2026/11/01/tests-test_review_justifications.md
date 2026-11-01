---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_review_justifications.py

# `tests/test_review_justifications.py`

## 1. Purpose

This file is the test suite for the `reasons.review_justifications` module. That module's job is to take a set of derived beliefs (nodes whose justification is an `SL` — "source list" — with multiple antecedents) and ask an LLM whether those antecedents should be linked as **ALL** (all required), **ANY** (any one suffices), or **MIXED** (some required, some independent). The test file validates three layers of that pipeline:

| Layer | Function Under Test | Responsibility |
|---|---|---|
| Parsing | `parse_justification_review` | Turn a raw LLM string (possibly with prose) into a clean list of dicts |
| Filtering | `_has_multi_antecedent_sl` | Decide whether a node is eligible for review |
| Orchestration | `review_justifications` | Batch nodes, call the model, collect results |
| API | `api.review_justifications` | Public entry point: load from DB, run review, return summary stats |

The file owns the contract that "given these inputs, the justification reviewer produces exactly these outputs" without any real model call.

## 2. Key Components

### Fixture: `network_db`
Creates a throwaway SQLite DB (via `tempfile.mkdtemp()`) pre-populated with:
- Three premise nodes: `fact-a`, `fact-b`, `fact-c` (no SL justifications)
- `derived-1`: SL with `["fact-a", "fact-b"]` (2 antecedents)
- `derived-2`: SL with `["fact-a", "fact-b", "fact-c"]` (3 antecedents)
- `single-ant`: SL with `["fact-a"]` (1 antecedent)

This gives every test in `TestApiReviewJustifications` a realistic graph to query.

### `TestParseJustificationReview`
Tests `parse_justification_review(response: str) -> list[dict]`. The contract it pins down:

- **Input**: a JSON array embedded possibly inside free-text prose.
- **Output**: a list of dicts, each guaranteed to have keys `id`, `classification`, `required_antecedents`, `independent_antecedents`, `comment`.
- **Defaults**: missing/null `classification` → `"ALL"`; missing/null list fields → `[]`; missing/null `comment` → `""`.
- **Normalization**: classification is uppercased (`"any"` → `"ANY"`).
- **Rejection**: items lacking an `id` key are silently dropped.
- **Graceful failure**: non-JSON input returns `[]` (no exception).

### `TestHasMultiAntecedentSl`
Tests `_has_multi_antecedent_sl(node: dict, min_antecedents: int) -> bool`.
- Returns `True` only when the node has at least one justification of type `"SL"` whose `antecedents` list length is **≥ `min_antecedents`**.
- `min_antecedents=2` means "at least 2 antecedents" (i.e., multi-antecedent).
- A node with no `justifications` key returns `False`.

### `TestReviewJustifications`
Tests the orchestration function `review_justifications(nodes, batch_size, parallel, min_antecedents)`:
- **Filtering**: only nodes with qualifying SL justifications are sent to the model (single-antecedent nodes and premises are excluded).
- **Parallelism**: `batch_size=1, parallel=2` produces two concurrent model calls (mocked via `side_effect`).
- **`min_antecedents`**: raising it to 3 excludes the 2-antecedent node.

### `TestApiReviewJustifications`
Tests the public `api.review_justifications(db_path, belief_ids)` entry point:
- Returns a summary dict with keys `reviewed`, `convert_any`, `keep_all`, `convert_mixed`.
- Supports filtering by `belief_ids`.
- **Critical invariant**: the call is **read-only** — the underlying node's justifications are unchanged after the call.

## 3. Patterns

- **LLM mocking via `patch`**: Every test that exercises model interaction patches `reasons.review_justifications.invoke_model` (not the real model). This keeps tests deterministic and fast. The `side_effect` variant in `test_parallel` simulates different responses per prompt to verify batching.
- **Layered test classes**: Each class isolates one function. You can read `TestParseJustificationReview` without needing to understand the batching logic, and vice-versa.
- **Fixture-scoped temp DB**: `network_db` uses `tempfile.mkdtemp()` so each test gets an isolated SQLite file. No shared state between tests.
- **Defensive parsing tests**: The parser tests explicitly cover the "LLM returned garbage" scenarios (surrounding prose, null fields, missing keys, non-JSON) — a pattern that acknowledges the LLM output is untrusted.
- **No `conftest.py` dependency beyond the fixture defined inline**: The fixture is local to this file, not shared.

## 4. Dependencies

### This file imports
| Import | Used For |
|---|---|
| `reasons.api` | `add_node`, `show_node` — building and inspecting the test DB |
| `reasons.review_justifications.parse_justification_review` | Parser under test |
| `reasons.review_justifications.review_justifications` | Orchestration under test |
| `reasons.review_justifications._has_multi_antecedent_sl` | Filter under test (private, but tested directly) |
| `unittest.mock.patch` | Replacing `invoke_model` with canned JSON |
| `pytest` | Test runner, fixtures, `assert` |

### What depends on this file
- **Nothing imports a test file.** It is a leaf in the dependency graph.
- However, it *constrains* `reasons/review_justifications.py` and `reasons/api.py`: any change to their signatures, return shapes, or default values must keep these assertions green.

### Note on the dual package layout
The repo has both `reasons/` and `reasons_lib/` with identical module names. The tests import from `reasons`. If `pyproject.toml` maps `reasons` → `reasons_lib`, the tests still exercise the same code. A developer should check `pyproject.toml` to confirm which tree is canonical.

## 5. Flow

```
Test execution for TestApiReviewJustifications.test_returns_summary:

1. Fixture network_db runs:
   - Creates temp dir → path/test.db
   - Calls api.add_node 6 times to populate the graph

2. Test body:
   - Builds mock_response (JSON string with 2 results)
   - Patches reasons.review_justifications.invoke_model
   - Calls api.review_justifications(db_path=network_db)

   Inside api.review_justifications (implementation, not in this file):
   a. Load all IN nodes from the DB
   b. Filter via _has_multi_antecedent_sl(node, min_antecedents=2)
      → keeps derived-1 (2 ant.), derived-2 (3 ant.)
      → drops single-ant (1 ant.), fact-a/b/c (no SL)
   c. Batch the kept nodes, build a prompt per batch
   d. Call invoke_model(prompt)  ← intercepted by patch, returns mock_response
   e. Call parse_justification_review(mock_response)
      → extracts JSON from string, normalizes fields
   f. Aggregate: count by classification → summary dict

3. Assertions:
   - result["reviewed"] == 2
   - result["convert_any"] == 1   (derived-1 → ANY)
   - result["keep_all"] == 1      (derived-2 → ALL)
   - result["convert_mixed"] == 0
```

For the **parallel** test, the flow branches:
```
batch_size=1, parallel=2
  → 2 batches, each 1 node
  → invoke_model called twice (side_effect routes by prompt content)
  → results merged → 2 entries
```

## 6. Invariants

These are the rules the tests **enforce** (and the implementation must uphold):

1. **Classification default is `"ALL"`** — if the LLM omits or nulls the field, the node is treated as requiring all antecedents (the conservative/safe choice). *(tests: `test_parse_defaults_classification_to_all`, `test_parse_null_fields`)*

2. **Classification is always uppercase** — `"any"`, `"All"`, etc. are normalized. Downstream code can rely on exact-match against `"ANY"`, `"ALL"`, `"MIXED"`. *(test: `test_parse_normalizes_case`)*

3. **Items without an `"id"` are dropped, not raised** — a malformed LLM array element is silently skipped rather than crashing the batch. *(test: `test_parse_skips_items_without_id`)*

4. **The review is read-only** — `api.review_justifications` must not mutate the stored justifications. The test `test_does_not_modify_db` explicitly re-reads the node and asserts the original 2-antecedent SL is intact. *(test: `test_does_not_modify_db`)*

5. **Only SL-type justifications with ≥ `min_antecedents` are eligible** — premises (empty justifications) and single-antecedent nodes are never sent to the model. *(tests: `test_filters_to_multi_antecedent`, `test_respects_min_antecedents`)*

## 7. Error Handling

The error-handling strategy here is **defensive parsing, not exception raising**:

- **Non-JSON LLM response**: `parse_justification_review("no json here")` returns `[]` instead of raising `json.JSONDecodeError`. The test `test_parse_empty_response` locks this in. This means a total LLM failure degrades to "no reviews" rather than a crash.
- **Null fields in JSON**: `null` values for `classification`, lists, or `comment` are coalesced to their defaults (`"ALL"`, `[]`, `""`). The test `test_parse_null_fields` verifies every field.
- **Missing `id`**: the item is filtered out. No partial results are emitted.
- **What is NOT handled here**: there are no tests for network timeouts, model rate-limiting, or DB I/O failures. Those are presumably handled (or not) in `invoke_model` and `api` respectively, outside this test's scope. A developer adding resilience should add those tests alongside the implementation.
- **`MagicMock` is imported but never used** — a minor cleanup opportunity.

---

## Topics to Explore

- [file] `reasons/review_justifications.py` — The implementation under test. Understanding how `invoke_model` is called, how prompts are constructed, and how `parallel` batching works will reveal what these tests actually constrain.
- [file] `reasons/api.py` — The public API layer that `api.review_justifications` lives in. Shows how DB loading, node filtering, and the summary-dict construction are wired together.
- [file] `tests/test_review.py` — Sibling test for `reasons/review.py`. Comparing the two reveals whether `review` and `review_justifications` share patterns or diverge in how they interact with the LLM.
- [general] `all-vs-any-justification-semantics` — The core domain concept: what it means for a derived belief's antecedents to be ALL-required vs. ANY-sufficient, and how `MIXED` splits them. Understanding this shapes how you read both the tests and the prompt templates.
- [file] `pyproject.toml` — Clarifies the `reasons` vs. `reasons_lib` package relationship, test runner configuration, and any pytest plugins in use.

## Beliefs

- `review-justifications-is-read-only` — `api.review_justifications` never mutates the stored SL justifications; it returns a summary dict of proposed classifications, leaving persistence to a separate step.
- `parse-justification-review-defaults-to-all` — When the LLM omits or nulls the `classification` field, the parser assigns `"ALL"`, ensuring the conservative "require every antecedent" semantic is the fallback.
- `min-antecedents-gates-eligibility` — A node is only sent to the LLM for review if it has at least one SL justification whose antecedent count is ≥ `min_antecedents` (default 2); single-antecedent nodes and premises are never reviewed.
- `invoke-model-is-the-mock-boundary` — All LLM interaction in `review_justifications` flows through a single `invoke_model` function, which is the sole patch point in every test; no other model-calling path exists in the module.
- `classification-is-always-uppercase` — After `parse_justification_review`, the `classification` field is guaranteed to be one of `"ALL"`, `"ANY"`, or `"MIXED"` in uppercase, enabling exact-string matching in downstream logic.
