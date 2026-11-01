---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_classify_defeaters.py

# `tests/test_classify_defeaters.py`

## Purpose

This file tests the **defeater-reason classification** pipeline — the subsystem that uses an LLM to label *why* one node (the defeater) defeated another node (the defeated justification). The project maintains a belief/argument graph where nodes can "defeat" each other's justifications, and this classification step assigns a structured reason code (e.g., `scope-mismatch`, `over-generalizes`) to that defeat event so downstream queries, reports, and repairs can reason about the *type* of failure.

The file owns two responsibilities:

1. **Unit-testing `classify_defeat_reason`** — the low-level function that sends a prompt to an LLM and parses the response into a known type string.
2. **Integration-testing `api.classify_defeat_reason_types`** — the batch API that walks the database, finds unclassified defeaters, calls the LLM for each, and (optionally) writes the result into node metadata.

## Key Components

| Symbol | Kind | Contract |
|--------|------|----------|
| `TestClassifyDefeatReason` | Test class | Verifies `classify_defeat_reason(defeater_text, defeated_text, model, timeout) -> str`. Returns a valid type on success, `""` on failure. |
| `TestClassifyDefeatReasonTypes` | Test class | Verifies `api.classify_defeat_reason_types(model, dry_run, db_path, defeater_type_filter)` returns a dict with keys `"classified"`, `"skipped"`, `"errors"`. In dry-run mode, no DB write occurs. |
| `TestDefeatReasonTypes` | Test class | Asserts the `DEFEAT_REASON_TYPES` constant has exactly 7 entries, all kebab-case (contain `"-"`). |
| `DEFEAT_REASON_TYPES` | Constant (imported) | The closed set of valid classification labels. The LLM must map its free-text answer into this set. |
| `_setup_db` (helper) | Method | Creates a minimal graph: premise `p1` → defeater `d1` with `defeater_type="migrated-retraction"`. Returns the DB path string. |

### The LLM mock pattern

Every test that exercises the LLM path uses this idiom:

```python
mock_result = type("R", (), {
    "returncode": 0, "stdout": "unsupported-conjunct", "stderr": ""})()
with patch("reasons.llm.shutil.which", return_value="/usr/bin/claude"), \
     patch("reasons.llm.subprocess.run", return_value=mock_result):
```

This replaces both the *binary-existence check* (`shutil.which`) and the *actual subprocess call* inside `reasons.llm`. Using `type("R", (), {...})()` instead of `MagicMock` keeps the mock structurally identical to a `subprocess.CompletedProcess`, so attribute access (`returncode`, `stdout`, `stderr`) is guaranteed to work without `configure_mock` boilerplate.

## Patterns

- **Mock-at-the-subprocess boundary.** The tests never mock `classify_defeat_reason` itself; they mock one layer lower (`subprocess.run`), so the text-parsing / type-matching logic under test still executes. This catches regressions in the "extract a known token from prose" step.
- **Dry-run vs. apply duality.** `test_dry_run_classifies_without_writing` and `test_apply_writes_metadata` form a pair proving the `dry_run` flag is the sole gate on persistence. The dry-run test explicitly asserts `"defeat_reason_type" not in node["metadata"]` after the call.
- **Idempotency via "already classified" skip.** `test_skips_already_classified` calls the API twice and asserts the second call produces zero classified items and a skip reason of `"already classified"`.
- **Filtering by `defeater_type`.** `test_filters_by_defeater_type` verifies that a `defeater_type_filter` that doesn't match results in a skip, while a matching filter proceeds.
- **Negative-path coverage.** Empty DB (no defeaters), LLM returning unparseable text, LLM subprocess raising an exception — all are exercised.

## Dependencies

```
test_classify_defeaters.py
├── reasons.api                    (init_db, add_node, defeat_justification, show_node, classify_defeat_reason_types)
├── reasons.review                 (classify_defeat_reason, DEFEAT_REASON_TYPES)
├── reasons.llm                    (indirectly — mocked via shutil.which / subprocess.run)
└── unittest.mock.patch            (standard library)
```

Nothing in the repo imports this test file (it's a leaf). Its production counterparts (`reasons/review.py`, `reasons/api.py`) are the modules under test.

## Flow

### `classify_defeat_reason` (low-level)

```
defeater_text + defeated_text + model + timeout
        │
        ▼
  reasons.llm: shutil.which(model)  ──►  None?  → return ""
        │
        ▼
  subprocess.run([...], timeout=timeout)
        │
        ▼  (exception? → return "")
  stdout string
        │
        ▼
  Match against DEFEAT_REASON_TYPES:
    • exact token present in stdout  → return that token
    • no match                      → return ""
```

The tests confirm all three branches: exact match (`test_returns_valid_type`), extraction from prose (`test_extracts_type_from_prose`), and no-match (`test_returns_empty_on_unrecognized`).

### `api.classify_defeat_reason_types` (batch API)

```
1. Open DB, query all nodes that ARE defeaters
   (skip non-defeater nodes — test_skips_non_defeater_nodes)

2. For each defeater node:
   a. If metadata already has "defeat_reason_type" → add to "skipped"
   b. If defeater_type_filter set and node's type doesn't match → add to "skipped"
   c. Else call classify_defeat_reason(...)
      - success → add to "classified"; if not dry_run, write metadata
      - failure ("" returned) → add to "errors"

3. Return {"classified": [...], "skipped": [...], "errors": [...]}
```

## Invariants

- **Closed vocabulary.** `DEFEAT_REASON_TYPES` is a fixed set of exactly 7 kebab-case strings. The parser only accepts one of these; anything else is treated as a failure.
- **Idempotency.** A defeater that already carries `defeat_reason_type` in its metadata is never re-classified.
- **Dry-run purity.** When `dry_run=True`, no mutation of the database occurs — verified by re-reading the node and asserting the key is absent.
- **Defeater-only scope.** Only nodes that are actual defeaters (created via `defeat_justification`) are eligible. Plain premise nodes in the DB produce zero results.
- **Filter is exact-match on `defeater_type`.** The `defeater_type_filter` parameter compares against the node's `defeater_type` field; non-matching nodes are skipped with a descriptive reason.

## Error Handling

| Failure mode | Where handled | Observed behaviour |
|---|---|---|
| LLM binary not found (`shutil.which` → `None`) | Inside `reasons.llm` | `classify_defeat_reason` returns `""`; API puts the node in `"errors"` |
| Subprocess raises (timeout, crash) | `try/except` around `subprocess.run` in `reasons.llm` | Returns `""`; API puts the node in `"errors"` |
| LLM returns unparseable prose | Type-matching step in `classify_defeat_reason` | Returns `""`; API puts the node in `"errors"` |
| Node already classified | `api.classify_defeat_reason_types` pre-check | Node goes to `"skipped"` with reason `"already classified"` |
| Filter mismatch | `api.classify_defeat_reason_types` pre-check | Node goes to `"skipped"` with reason containing the mismatched type name |

The design is **fail-soft**: no exception propagates to the caller of `classify_defeat_reason`. The API-level function aggregates failures into the `"errors"` list rather than aborting the batch.

---

## Topics to Explore

- [file] `reasons/review.py` — Contains `classify_defeat_reason` and `DEFEAT_REASON_TYPES`; you'll see the prompt construction, the token-extraction regex/loop, and the exact 7-type vocabulary.
- [function] `reasons/api.py:classify_defeat_reason_types` — The batch orchestrator this test exercises; shows how it queries defeaters, applies the filter, calls the LLM, and writes metadata.
- [file] `reasons/llm.py` — The thin subprocess wrapper (`shutil.which` + `subprocess.run`) that every LLM call flows through; understanding its interface explains why the mocks target those two symbols.
- [file] `tests/test_migrate_defeaters.py` — Likely tests the migration path that sets `defeater_type="migrated-retraction"` (the type used in `_setup_db`), giving context on how legacy defeaters enter the system.
- [general] `defeater-reason-vocabulary` — The 7 reason types (`scope-mismatch`, `over-generalizes`, `unsupported-conjunct`, etc.) form the project's ontology for *why* an argument fails; exploring where they're consumed (reports, repair, review) reveals their downstream impact.

---

## Beliefs

- `classify-defeat-reason-returns-empty-string-on-failure` — `classify_defeat_reason` never raises; every failure path (missing binary, subprocess exception, unparseable output) returns `""`, letting callers distinguish "no classification" from an exception.
- `defeat-reason-types-is-closed-seven-element-set` — `DEFEAT_REASON_TYPES` contains exactly 7 kebab-case strings; the parser is a membership check against this set, so any future type must be added here before the LLM prompt can reference it.
- `dry-run-classify-mutates-nothing` — When `dry_run=True`, `api.classify_defeat_reason_types` reads the DB but performs zero writes; the node's `metadata` dict is provably unchanged after the call.
- `already-classified-nodes-are-skipped-not-overwritten` — A defeater that already has `defeat_reason_type` in its metadata is placed in the `"skipped"` bucket and is never re-sent to the LLM, making the batch operation idempotent.
- `llm-mock-targets-subprocess-boundary` — All LLM-dependent tests patch `reasons.llm.shutil.which` and `reasons.llm.subprocess.run` specifically, meaning the parsing/matching logic in `classify_defeat_reason` is always exercised for real while the external process call is replaced.
