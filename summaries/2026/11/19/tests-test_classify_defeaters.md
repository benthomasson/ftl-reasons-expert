---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_classify_defeaters.py

## Explanation of `tests/test_classify_defeaters.py`

### 1. Purpose

This file is the test suite for the **classify-defeaters** feature. That feature has two layers, and both are covered here:

1. A pure, single-item classifier: `reasons.review.classify_defeat_reason(...)` — asks an LLM to label a defeater with one of a fixed vocabulary of defeat-reason types.
2. A batch/orchestration command: `reasons.api.classify_defeat_reason_types(...)` — walks defeater nodes in the DB, calls the classifier, and either reports results (dry-run) or writes `defeat_reason_type` into node metadata (apply).

The file also asserts the vocabulary contract via `TestDefeatReasonTypes`.

It owns the responsibility of locking down:
- the return contract of `classify_defeat_reason` (type string or `""`),
- the result-dict contract of `classify_defeat_reason_types` (`classified` / `skipped` / `errors`),
- the dry-run vs. apply semantics (no writes vs. metadata writes),
- skip/filter behavior (`already classified`, `defeater_type_filter`),
- the exact shape of `DEFEAT_REASON_TYPES`.

### 2. Key Components

**`TestClassifyDefeatReason`** — unit tests for `classify_defeat_reason(defeater_text, defeated_text, model, timeout)`.
- `test_returns_valid_type`: exact stdout `"unsupported-conjunct"` → returns it verbatim.
- `test_extracts_type_from_prose`: stdout `"The failure mode is over-generalizes because..."` → parser extracts `"over-generalizes"` from surrounding prose.
- `test_exact_match_preferred`: stdout `"scope-mismatch"` → returns `"scope-mismatch"` (name signals that a whole-output exact match is preferred over fuzzy/prose extraction).
- `test_returns_empty_on_unrecognized`: stdout `"I cannot classify this"` → returns `""` (no matching type token).
- `test_returns_empty_on_error`: `subprocess.run` raises `Exception("timeout")` → returns `""`. No exception escapes.

The two `patch` targets are `reasons.llm.shutil.which` (so the LLM wrapper believes the `claude` CLI is on PATH) and `reasons.llm.subprocess.run` (so no real subprocess runs). This means `classify_defeat_reason` internally routes through `reasons.llm` and shells out to a model CLI.

**`TestClassifyDefeatReasonTypes`** — integration tests for `api.classify_defeat_reason_types(model, dry_run, db_path, defeater_type_filter=...)`.
- `_setup_db(tmp_path)` helper seeds a throwaway DB: `init_db`, `add_node("p1")`, `add_node("d1", sl="p1")`, then `defeat_justification("d1", 0, "...", defeater_type="migrated-retraction")`. That last call creates a defeater node — the thing under test. Returns the DB path as a string.
- `test_dry_run_classifies_without_writing`: `dry_run=True` → `result["classified"]` has one entry with `id` and `defeat_reason_type`; `api.show_node(...)["metadata"]` does **not** contain `defeat_reason_type`.
- `test_apply_writes_metadata`: `dry_run=False` → same entry appears, and `show_node(...)["metadata"]["defeat_reason_type"] == "scope-mismatch"`.
- `test_skips_already_classified`: apply once, then dry-run again → second run has `classified == []` and a `skipped` entry whose `reason` is `"already classified"`. Confirms the "done" marker is the presence of metadata.
- `test_filters_by_defeater_type`: with `defeater_type_filter="invalid-inference"`, the `migrated-retraction` defeater is skipped and the skip reason mentions `migrated-retraction`; with `defeater_type_filter="migrated-retraction"`, it is classified. So the filter works on the *existing* `defeater_type` metadata, not on the new `defeat_reason_type`.
- `test_skips_non_defeater_nodes`: DB with only two plain premise nodes → `classified == []`. Non-defeaters are ignored, and importantly no LLM mock is installed, so this also demonstrates the code never invokes the LLM when there is nothing to classify.
- `test_errors_on_classification_failure`: LLM returns unclassifiable prose → `classified == []` **and** `len(result["errors"]) == 1`. This is the key contrast with the pure function: it returns `""`, and the batch layer interprets that as an error record rather than silently discarding it.

**`TestDefeatReasonTypes`**
- `test_all_types_valid`: `len(DEFEAT_REASON_TYPES) == 7`, and every entry contains `-` (kebab-case). Examples seen in tests: `unsupported-conjunct`, `over-generalizes`, `scope-mismatch`. The set is a closed vocabulary — the parser can only return members of it.

**Result-dict contract of `classify_defeat_reason_types`** (inferred from assertions):
```
{
  "classified": [ {"id": <defeater node id>, "defeat_reason_type": <type>} , ... ],
  "skipped":    [ {"reason": <text>}, ... ],
  "errors":     [ ... ],
}
```
Only `classified[*].id`, `classified[*].defeat_reason_type`, `skipped[*].reason`, and `len(errors)` are asserted, so those are the load-bearing shape.

### 3. Patterns

- **Mock-at-import-site**: `patch("reasons.llm.shutil.which", ...)` and `patch("reasons.llm.subprocess.run", ...)` — patching where names are looked up (`reasons.llm`), not where they originate (`shutil`/`subprocess`). This pins the architectural boundary: the classifier depends on `reasons.llm` as its subprocess seam.
- **Ad-hoc result stubs via `type("R", (), {...})()`**: instead of `MagicMock`, each test builds a tiny object exposing `returncode`, `stdout`, `stderr`. Matches the shape `subprocess.run(..., capture_output=True)` would produce and keeps assertions readable.
- **Seeded-DB fixture helper**: `_setup_db` centralizes the "premise + belief + defeater" scaffold so each integration test is one line of setup. Uses pytest's `tmp_path` for filesystem isolation.
- **Dry-run / apply duality**: the same API is exercised twice with different `dry_run` flags to test both observational and mutating behavior. This is an idempotency and safety pattern — dry-run must be observationally transparent.
- **Skip-with-reason taxonomy**: each `skipped` entry carries a human-readable `reason` string (`"already classified"`, text containing the filter type). Tests assert on substrings of that string, so the message wording is part of the contract.

### 4. Dependencies

Imports in this file:
- `from unittest.mock import patch`
- `from reasons import api` — `init_db`, `add_node`, `defeat_justification`, `show_node`, `classify_defeat_reason_types`
- `from reasons.review import classify_defeat_reason, DEFEAT_REASON_TYPES`

Indirect dependencies exercised (via patching):
- `reasons.llm` — imports `shutil` and `subprocess` and drives the model CLI (`claude` in tests). The `shutil.which` check confirms the wrapper refuses to run when the binary is not installed.
- The DB layer behind `api.*` (likely `reasons.storage`/`reasons.pg`), mediated by `db_path` — the tests only use the local file path.

No module imports this test file. Related test files in the repo: `test_defeat_justification.py`, `test_defeat_with_scope.py`, `test_migrate_defeaters.py` — the last is likely the source of the `migrated-retraction` defeater type used here.

### 5. Flow

**Pure-function path** (`TestClassifyDefeatReason`):
1. Build a stub `subprocess.CompletedProcess`-like object.
2. Install `shutil.which` → `/usr/bin/claude` and `subprocess.run` → stub.
3. Call `classify_defeat_reason("defeater text", "defeated text", "claude", 300)`.
4. Inside, `reasons.llm` sees the binary exists, runs a mocked subprocess, receives stdout, parses it: try exact match against `DEFEAT_REASON_TYPES`, else scan for a member token in the prose, else return `""`.
5. On any exception inside the LLM call, the function swallows it and returns `""`.

**Batch path** (`TestClassifyDefeatReasonTypes`):
1. `_setup_db` seeds a DB and creates a defeater node via `defeat_justification(..., defeater_type="migrated-retraction")`.
2. Call `api.classify_defeat_reason_types(model="claude", dry_run=..., db_path=db, defeater_type_filter=...)`.
3. The API enumerates defeater nodes, filters by `defeater_type_filter` and by "already has `defeat_reason_type` metadata" (skipping with reasons).
4. For each remaining defeater, it calls the classifier (which reaches the mocked LLM).
5. Non-empty type → add `{"id", "defeat_reason_type"}` to `classified`; if `dry_run=False`, also write `metadata["defeat_reason_type"]`.
6. Empty type → append to `errors`.
7. Return the aggregate dict.

### 6. Invariants

- `classify_defeat_reason` **never raises** for LLM failure or unparseable output — it returns `""`.
- The returned type is always a member of `DEFEAT_REASON_TYPES` (or `""`).
- `DEFEAT_REASON_TYPES` has exactly **7** entries, all kebab-case (contain `-`).
- `dry_run=True` must be **read-only**: no node metadata is mutated.
- `dry_run=False` sets `metadata["defeat_reason_type"]` on the defeater node.
- Presence of `metadata["defeat_reason_type"]` is the "already done" signal; re-running skips those nodes with reason `"already classified"`. The batch operation is therefore idempotent on repeated apply.
- `defeater_type_filter`, when provided, excludes nodes whose existing `defeater_type` does not match, and the skip reason names the actual defeater_type.
- Non-defeater nodes are never classified (no LLM call attempted when the DB has none).
- In the batch layer, an unclassifiable defeater is surfaced in `result["errors"]`, **not** raised. The batch as a whole continues (this test has only one node, so "continues" is implied rather than proven).
- `classified[*].id` is the node id of the defeater, not the defeated node.

### 7. Error Handling

- **Inside `classify_defeat_reason`**: `subprocess.run` exceptions (e.g., timeout) are caught and converted to `""`. This is deliberate failure-softening — the pure function does not distinguish "timeout" from "unrecognized response".
- **Unrecognized LLM output**: also `""`. There is no fallback that guesses a type.
- **Inside `classify_defeat_reason_types`**: an empty classifier result is treated as an error, appended to `result["errors"]`, and the item is neither classified nor written.
- **Skips are not errors**: already-classified and filter-mismatched nodes go to `skipped`, never `errors`. `errors` is reserved for actual classification failure.
- **No exception assertions** anywhere in the file — the design is that failures become data in the result dict, not raised exceptions. A future regression that lets exceptions escape would fail these tests by error rather than by assertion.

---

## Topics to Explore

- [function] `reasons/review.py:classify_defeat_reason` — See exactly how stdout is parsed (exact match vs. first/substring match) and where the exception is swallowed.
- [function] `reasons/api.py:classify_defeat_reason_types` — Inspect the result-dict keys, the node enumeration query, and the metadata write path for `dry_run=False`.
- [file] `reasons/llm.py` — Understand the `shutil.which` gate and `subprocess.run` contract (returncode/stdout/stderr) that all these mocks emulate.
- [function] `reasons/api.py:defeat_justification` — How defeater nodes and their `defeater_type` metadata are produced, which is the input to the filter under test.
- [file] `tests/test_migrate_defeaters.py` — Where the `migrated-retraction` defeater type used by `_setup_db` originates.

## Beliefs

- `defeat-reason-types-has-seven-entries` — `DEFEAT_REASON_TYPES` contains exactly 7 members.
- `defeat-reason-types-are-kebab-case` — Every entry in `DEFEAT_REASON_TYPES` contains a hyphen.
- `classify-defeat-reason-returns-empty-on-failure` — The pure classifier returns `""` on subprocess exception and on unrecognized LLM output, never raising.
- `dry-run-does-not-write-metadata` — `classify_defeat_reason_types(..., dry_run=True)` never adds `defeat_reason_type` to node metadata.
- `apply-mode-writes-defeat-reason-type-metadata` — With `dry_run=False`, the classifier result is persisted at `node["metadata"]["defeat_reason_type"]`.
- `already-classified-defeaters-are-skipped` — Re-running classification skips nodes whose metadata already contains `defeat_reason_type`, reporting reason `"already classified"`.
- `classification-failures-go-to-errors-list` — Unclassifiable defeaters are collected in `result["errors"]` instead of being raised or written.
- `defeater-type-filter-matches-existing-metadata` — `defeater_type_filter` matches against the defeater node's existing `defeater_type`, not the new `defeat_reason_type`.
