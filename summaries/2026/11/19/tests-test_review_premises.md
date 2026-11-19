---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_review_premises.py

# `tests/test_review_premises.py`

## Purpose

This file is the test suite for the **review-premises** feature — an LLM-driven quality audit of a knowledge base's *premises* (observation / leaf beliefs) against the source documents they cite. It exercises four layers of the stack:

| Layer | Tested via |
|---|---|
| Prompt formatting | `format_premise_for_review` |
| LLM response parsing | `parse_premise_review_response` |
| Batch LLM orchestration | `review_premises` (module) |
| DB-backed API + persistence | `api.review_premises` |
| CLI wiring | `reasons.cli.cmd_review_premises` |

The file owns **no production logic** — it is a contract definition for the `reasons.review_premises` module and for the `review_premises` entry point in `reasons.api`.

---

## Key Components

### `TestFormatPremiseForReview`
Tests `format_premise_for_review(belief_id, nodes, source_content) -> str`.

- **Missing node** → returns `""` (empty string, not exception).
- **Happy path** → Markdown block:
  - `### {belief_id}` header,
  - the node's `text`,
  - a `Source reference:` line **only if** `source` is present on the node,
  - the provided `source_content` verbatim (the quoted passage the LLM must judge against).
- The `test_no_source_field` case is important: the `Source reference:` label must be omitted when no source path exists (the function must still emit the premise text and content).

### `TestParsePremiseReviewResponse`
Tests `parse_premise_review_response(response_text) -> list[dict]`.

Contract:
- Accepts LLM output that may include **prose before/after** a JSON array (`test_json_with_prose` proves it extracts the embedded array).
- Each element is a dict with defaults filled in:
  - `accurate` → `True`
  - `well_scoped` → `True`
  - `error_type` → `None`
  - `comment` → `""`
- Malformed input (no JSON) → `[]`.
- Preserves order and identity across multiple results.

### `TestReviewPremisesBatchLoop`
Tests `review_premises(nodes, ids, source_contents, model=..., on_batch=...) -> list[dict]`.

- Calls the underlying LLM via `reasons.llm` (patched `shutil.which` + `subprocess.run`).
- `on_batch` callback fires **once per completed batch** with that batch's results.
- Empty `ids` → `[]` without invoking anything.
- A raising batch is **swallowed** and produces no results for that batch; the run continues.

### `TestApiReviewPremises`
End-to-end tests of `api.review_premises(...)` against a real (temporary SQLite) DB built with `api.init_db` / `api.add_node` and read back with `api.export_network`.

Return dict shape exercised by tests:

- `results` — list of LLM result dicts (`id`, `accurate`, `well_scoped`, `error_type`, `comment`).
- `total_premises` — premise count *before* `sample`/`belief_ids` narrowing (2 observations in the fixture; `derived-1` excluded).
- `reviewed` — count actually sent to the LLM.
- `skipped_no_source` — premises dropped for lacking `source`.
- `inaccurate` — count of `accurate is False`.
- `overgeneralized` — count of `well_scoped is False`.

`test_specific_belief_ids` and `test_sample_limits` pin the narrowing semantics: `belief_ids` restricts which premises are reviewed; `sample` truncates the set.

### `TestCliDispatch`
Minimal smoke tests:
- `reasons.cli.cmd_review_premises` exists and is callable.

Note: `test_dispatch_table` imports `main` and `argparse` but never uses them — the test only asserts `callable(cli.cmd_review_premises)`. If dispatch-table behavior ever regresses, this test won't catch it.

---

## Patterns

1. **Class-scoped grouping by unit under test** — one `Test*` class per function/API boundary, making coverage gaps visible at a glance.

2. **Mocking at the `reasons.llm` boundary** — every LLM-touching test patches `reasons.llm.shutil.which` and `reasons.llm.subprocess.run`. This confirms the production code shells out to the `claude` binary, and lets tests run offline/deterministically.

3. **Ad-hoc `CompletedProcess` stand-in** —
   ```python
   mock_result = type("R", (), {"returncode": 0, "stdout": llm_response, "stderr": ""})()
   ```
   Used repeatedly instead of `subprocess.CompletedProcess`, avoiding a real subprocess.

4. **Real-DB fixture via `tmp_path`** — `db_with_premises` builds a disposable SQLite DB with two observations (`obs-1`, `obs-2`) and one derived belief (`derived-1` via `sl="obs-1"`), a clean 3-node graph for filter/skip/persist tests.

5. **Persist-and-reload assertion** — `test_stores_metadata` calls `api.export_network` after the review to inspect the persisted metadata, verifying behavior at the storage layer rather than via return values.

---

## Dependencies

**Imports (what this file uses):**
- `json` — building/parsing fake LLM payloads.
- `unittest.mock.patch` — target `reasons.llm.shutil.which` and `reasons.llm.subprocess.run`.
- `pytest` — `pytest.fixture`, `tmp_path`.
- `reasons.api` — `init_db`, `add_node`, `export_network`, `review_premises`.
- `reasons.review_premises` — `format_premise_for_review`, `parse_premise_review_response`, `review_premises`.
- `reasons.cli` — `cmd_review_premises` (and `main`).

**Imported by:** nothing — this is a leaf test file.

**Implicit coupling:** the patches target `reasons.llm`, so any refactor that stops importing `shutil`/`subprocess` in `reasons.llm` will silently break every LLM-touching test (patch target missing → real binary invocation attempted).

---

## Flow

For each `TestApiReviewPremises` test the flow is:

```
tmp_path/
 ├─ test.db            ← api.init_db
 ├─ entry.md           ← source file on disk
obs-1 "PostgreSQL…" source=entry.md
obs-2 "Redis…"      source=entry.md
derived-1 "…"       sl=obs-1        (excluded: not a premise)
          │
          ▼
api.review_premises(belief_ids?, sample?, dry_run?, db_path=…)
          │
          ├─ load nodes from DB
          ├─ filter to premises  (sl is None)
          ├─ optional belief_ids narrow
          ├─ skip nodes with no source  → skipped_no_source++
          ├─ optional sample truncation
          ├─ read source file contents
          ├─ review_premises(nodes, ids, source_contents)
          │    └─ per-batch: format prompt → LLM (mocked) → parse
          │    └─ on_batch callback per batch
          ├─ if not dry_run:
          │      for each result: set metadata.premise_review_result
          │      (= error_type or "pass") + last_premise_reviewed
          └─ aggregate counts (reviewed, inaccurate, overgeneralized)
                     │
                     ▼
              return {"results", "total_premises", "reviewed",
                      "skipped_no_source", "inaccurate", "overgeneralized"}
```

The batch loop inside `reasons.review_premises` is only visible through `on_batch` and the aggregate list — the batch-size constant and prompt template are **not** asserted here, so they can drift without a test failure.

---

## Invariants

1. **Premises are leaves.** `api.review_premises` operates only on nodes with no `sl` (justification) — `derived-1` is never in `results` (`test_filters_premises_only`). This depends on the model in which a node with `sl` is a *derived* belief, and a node without `sl` is a premise/observation.

2. **No `source` → skipped.** Any premise without a `source` attribute is excluded before the LLM is invoked and counted in `skipped_no_source` (`reviewed == 0` in that case).

3. **Defaults fail-open on accuracy.** If the LLM omits `accurate` or `well_scoped`, defaults are `True`. A missing field will therefore be scored as *pass*, not *fail* (`test_defaults_for_missing_fields`).

4. **`dry_run=True` is a strict no-op on the DB.** After a dry run, `premise_review_result` must be absent from the node's metadata (`test_dry_run_no_metadata`, which also uses `.get("metadata", {})` so a missing metadata block is tolerated).

5. **Persisted metadata contract.** A non-dry run writes `metadata.premise_review_result = error_type or "pass"` and sets a `last_premise_reviewed` timestamp key (`test_stores_metadata`).

6. **`on_batch` is per-batch, not per-result.** With a single id it is invoked once with a 1-element list; the test asserts `batches == [1]`.

7. **`reviewed` is post-filter, post-sample, post-belief-ids.** `test_return_counts` asserts `reviewed == 2` when both observations are sent, and `test_sample_limits` asserts `reviewed == 1` with `sample=1`.

8. **Counts are independent dimensions.** `inaccurate` tracks `accurate is False`, `overgeneralized` tracks `well_scoped is False` — a result can increment both, or neither.

---

## Error Handling

- **Malformed LLM JSON** → `parse_premise_review_response` returns `[]`; no exception, no partial-object leakage.
- **Missing node during formatting** → `format_premise_for_review` returns `""` (silent skip, no `KeyError`).
- **Failing LLM batch** (subprocess raises) → `review_premises` swallows the exception; result list excludes the failed batch. If all batches fail, the return is `[]` (`test_failed_batch_continues`).
- **Whole API call** does not raise in any of the tested failure modes; errors surface only as missing results. This makes the API safe to call from schedulers/CLI where a hard failure would abort a larger run.
- **The tests themselves don't assert anything about error *messages* or logs** — a failed batch is indistinguishable from a successful empty parse via the public return value. If observability is important, that's an area with no coverage here.

---

## Topics to Explore

- [file] `reasons/review_premises.py` — The implementation under test: batching strategy, prompt construction, `format_premise_for_review`, `parse_premise_review_response`, and the `on_batch` callback wiring.
- [function] `reasons/api.py:review_premises` — The DB facade: which nodes qualify as premises, how `belief_ids`/`sample` compose, and metadata schema.
- [file] `reasons/llm.py` — The `claude` CLI adapter; explains why tests patch `shutil.which` and `subprocess.run` and what `returncode`/`stdout` contract the batch loop relies on.
- [file] `tests/test_review_justifications.py` — Sibling audit feature; useful for spotting shared conventions (batch callbacks, metadata keys, default policy on missing LLM fields).
- [general] `premise-vs-derived-classification` — How `api.add_node` and the `sl` parameter distinguish premises from derived beliefs, since that classification determines the entire test fixture.

---

## Beliefs

- `review-premises-filters-to-leaves` — `api.review_premises` only reviews nodes with no `sl`, so derived beliefs such as `derived-1` never appear in `results`.
- `review-premises-skips-nodes-without-source` — Premises lacking a `source` attribute are excluded before LLM invocation and counted in `skipped_no_source`.
- `parse-defaults-to-accurate` — `parse_premise_review_response` defaults `accurate` and `well_scoped` to `True` when the LLM omits them.
- `dry-run-skips-metadata-persistence` — `api.review_premises(dry_run=True)` never writes `premise_review_result` into node metadata.
- `failed-llm-batch-is-swallowed` — A subprocess exception inside a batch is caught and produces no results for that batch without aborting the run.
- `premise-review-metadata-has-two-keys` — Non-dry runs persist both `premise_review_result` (error_type or `"pass"`) and `last_premise_reviewed` on each reviewed node.
