---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_defeat_with_scope.py

# `tests/test_defeat_with_scope.py`

## Purpose

This file is an integration/contract test module for the public `reasons.api` facade. It verifies the behavior of **justified defeaters that use “scope beliefs”**: nodes that record what a premise does and does not establish. When a scope belief shows that an antecedent does not establish some required property, `defeat_with_scope` creates a derived defeater from those scope beliefs and uses it to defeat a specific justification of a target node.

The file owns tests for:

- Creation of scope-belief nodes as `IN` premises.
- Creation of a derived defeater whose `SL` justification depends on those scope beliefs.
- Truth-value changes: target becomes `OUT`, dependents cascade to `OUT`.
- Truth maintenance: challenging/retracting a scope belief makes the defeater `OUT` and restores the target.
- Persistence: export/import roundtrip preserves the scope beliefs, defeater, and defeated target state.
- Validation/error contracts.
- Unique generated IDs across multiple defeats.
- `defeat_reason_type` metadata for both `defeat_with_scope` and `defeat_justification`.

It does not test internal storage functions directly; it drives the public API against temporary SQLite databases.

## Key Components

### Imports

- `pytest` — used for `tmp_path` and `pytest.raises`.
- `from reasons import api` — all operations go through the public `reasons.api` module.
- `import json` — imported inside `test_defeat_with_scope_in_export` to write an export file for re-import.

### API surface exercised

The tests use these `reasons.api` functions:

- `api.init_db(db_path)` — initializes a fresh database.
- `api.add_node(node_id, text, sl=..., db_path=...)` — creates premise or derived nodes. `sl` is a comma-separated support-list string, e.g. `"p1,p2"`.
- `api.show_node(node_id, db_path=...)` — returns node data including `truth_value`, `justifications`, `metadata`, and `text`.
- `api.defeat_with_scope(...)` — core function under test. The calls imply a signature like:
  ```python
  defeat_with_scope(
      node_id,
      justification_index,
      scope_findings,
      missing_property,
      defeater_id=None,
      defeat_reason_type="",
      db_path=...,
  )
  ```
- `api.retract_node(node_id, db_path=...)` — used to challenge a scope belief.
- `api.export_network(db_path=...)` — returns a serializable network payload.
- `api.import_json(path, db_path=...)` — imports an exported JSON network.
- `api.add_justification(node_id, sl=..., db_path=...)` — adds a second justification in the uniqueness test.
- `api.defeat_justification(...)` — tested only for `defeat_reason_type` metadata.

### Test functions

- `test_defeat_with_scope_basic`
  - Builds `belief` from `premise-a,premise-b`.
  - Calls `defeat_with_scope` with two scope findings, each saying an antecedent establishes one property but not `"traceability"`.
  - Asserts result keys: `node_id`, `justification_index`, two `scope_belief_ids`, and `"belief"` in `changed`.
  - Asserts target truth value moves from `IN` to `OUT`.

- `test_scope_beliefs_are_premises`
  - Verifies each scope belief is an `IN` premise.
  - Asserts `justifications == []`.
  - Asserts metadata:
    - `metadata["scope_of"]` equals the antecedent, e.g. `"p1"`.
    - `metadata["for_defeater"]` equals the generated `defeater_id`.
  - Asserts scope-belief text contains `"establishes X"` and `"does not establish Y"`.

- `test_defeater_is_derived`
  - Verifies the defeater is an `IN` derived node.
  - It has exactly one justification.
  - That justification has `type == "SL"` and `antecedents == result["scope_belief_ids"]`.
  - Metadata records:
    - `metadata["defeats_node"] == "d1"`
    - `metadata["defeats_justification"] == 0`

- `test_challenge_scope_restores_target`
  - Defeats `target` using two scope beliefs.
  - Target initially becomes `OUT`.
  - Calls `retract_node` on one scope belief.
  - Asserts the scope belief becomes `OUT`, the defeater becomes `OUT`, and the original `target` returns to `IN`.

- `test_defeat_with_scope_in_export`
  - Defeats `d1`, exports the network, and checks the scope belief and defeater are present in `export["nodes"]`.
  - Checks the defeater’s exported justification has `antecedents == [scope_id]`.
  - Writes the export to JSON, imports it into a second database, and asserts `d1` is `OUT` there.

- `test_defeat_with_scope_errors`
  - Pins exception contracts:
    - Missing target node: `KeyError` matching `"not found"`.
    - Target with no justifications: `ValueError` matching `"no justifications"`.
    - Out-of-range justification index: `IndexError` matching `"out of range"`.
    - Empty `scope_findings`: `ValueError` matching `"scope_findings must not be empty"`.

- `test_scope_ids_are_unique`
  - Performs two separate defeats on `d1`, using justification indices `0` and `1`.
  - Passes explicit defeater IDs `"defeater-1"` and `"defeater-2"`.
  - Asserts the union of all generated `scope_belief_ids` has length 2, so scope IDs do not collide across defeats.

- `test_cascade_through_scope_defeat`
  - Builds `d1` from `p1` and `d2` from `d1`.
  - Defeats `d1` via scope beliefs.
  - Asserts both `d1` and its dependent `d2` become `OUT`.

- `test_defeat_reason_type_stored`
  - Passes `defeat_reason_type="unsupported-conjunct"`.
  - Asserts the result returns it and the defeater metadata stores it.

- `test_defeat_reason_type_omitted`
  - Omits `defeat_reason_type`.
  - Asserts result value is `""` and the key is absent from defeater metadata.

- `test_defeat_justification_reason_type`
  - Tests the separate `defeat_justification` API.
  - Calls it with a reason text `"overclaims scope"` and `defeat_reason_type="scope-mismatch"`.
  - Asserts the result and defeater metadata both contain `"scope-mismatch"`.

## Patterns

- **Black-box API testing**: the tests go through `reasons.api` rather than importing storage or graph internals.
- **Temporary database isolation**: every test uses `tmp_path` to create a unique SQLite database, so tests do not share state.
- **Arrange / Act / Assert**: each test seeds nodes, invokes one operation, then asserts result structures and resulting node states.
- **Truth-value assertions**: expected states are stringly typed as `"IN"` and `"OUT"`.
- **Metadata contract testing**: scope beliefs and defeaters are not just checked for existence; their metadata keys and values are asserted.
- **Export/import roundtrip**: the export test verifies that scope beliefs, defeater justifications, and defeated truth state survive serialization.
- **Exception contract testing**: `pytest.raises(..., match=...)` pins both exception types and message fragments.
- **Optional caller-supplied IDs**: `defeater_id` can be provided for deterministic tests, while scope IDs are still generated uniquely.
- **Conditional defeat semantics**: a scope belief is an asserted premise, not automatically derived fact. If it is retracted, the defeater loses support and the original target can be restored.

## Dependencies

### Direct dependencies

- `pytest` — test runner, `tmp_path`, exception assertions.
- `reasons.api` — the public API being tested.
- `json` — standard library, used for the export/import roundtrip test.

### Implementation dependencies implied by behavior

The tests indirectly exercise:

- Database initialization and node storage.
- Justification creation and lookup.
- Truth-value propagation / dependency graph maintenance.
- Metadata persistence.
- Export/import serialization format.
- Defeater creation logic for both scoped and non-scoped defeat.

### Reverse dependencies

This file is a test module. It is discovered by pytest and is not imported by production code. Changes to `reasons/api.py`, storage schema, export format, or truth-maintenance behavior can break this file.

## Flow

A typical test follows this sequence:

1. Create a temporary database path:
   ```python
   db = tmp_path / "test.db"
   api.init_db(str(db))
   ```
2. Add premise/derived nodes with `api.add_node`.
3. Call `api.defeat_with_scope(...)` with one or more `scope_findings`.
4. Inspect the returned dictionary:
   - `node_id`
   - `justification_index`
   - `scope_belief_ids`
   - `defeater_id`
   - `changed`
   - `defeat_reason_type`
5. Inspect created nodes with `api.show_node`.
6. In follow-up tests, either:
   - retract a scope belief and verify restoration,
   - export/import and verify persistence,
   - add another justification and verify ID uniqueness,
   - or assert exception behavior.

Conceptually, `defeat_with_scope` transforms scope findings into premise nodes. Those scope beliefs are then used as the antecedents of an `SL`-justified defeater. The defeater is linked by metadata to the target node and justification index. Because the target’s justification is defeated, the target becomes `OUT`; dependents cascade. If a scope belief is retracted, the defeater becomes `OUT`, which can restore the target.

The error tests also imply validation ordering: missing-target, no-justification, index-range, and empty-findings errors fire before the code fully processes scope findings. For example, several error calls provide scope findings without `does_not_establish`, but the test expects earlier validation errors instead of a missing-key error.

## Invariants

- A fresh database is initialized per test.
- A target node must exist before `defeat_with_scope` can succeed.
- The target must have at least one justification.
- `justification_index` must be valid for the target’s justifications.
- `scope_findings` must not be empty.
- Each scope finding normally contains:
  - `antecedent`
  - `establishes`
  - `does_not_establish`
- One scope-belief node is created per scope finding.
- Scope beliefs are `IN` premises with no justifications.
- Scope-belief metadata includes `scope_of` and `for_defeater`.
- The generated defeater is an `IN` derived node.
- The defeater has exactly one `SL` justification.
- The defeater’s `SL` antecedents are the returned `scope_belief_ids`.
- Defeater metadata includes `defeats_node` and `defeats_justification`.
- Defeating a target via scope beliefs sets the target to `OUT`.
- Dependents of a defeated node cascade to `OUT`.
- Retracting a supporting scope belief sets the scope belief and defeater to `OUT` and can restore the target to `IN`.
- Scope-belief IDs are unique across multiple defeats, even for the same target and antecedent.
- `defeat_reason_type` is stored only when non-empty.
- Exported networks include scope-belief and defeater nodes, and importing them preserves the defeated state.

## Error Handling

The test file asserts that errors are not swallowed:

- Missing node:
  ```python
  pytest.raises(KeyError, match="not found")
  ```
- Node with no justifications:
  ```python
  pytest.raises(ValueError, match="no justifications")
  ```
- Invalid justification index:
  ```python
  pytest.raises(IndexError, match="out of range")
  ```
- Empty `scope_findings`:
  ```python
  pytest.raises(ValueError, match="scope_findings must not be empty")
  ```

Successful paths do not use `try/except`; state changes are observed directly via `show_node`. The export/import test writes JSON to disk and imports it without testing import-error cases.

## Topics to Explore

- [function] `reasons/api.py:defeat_with_scope` — Implements scope-belief creation, derived defeater construction, metadata wiring, and validation behavior asserted here.
- [function] `reasons/api.py:defeat_justification` — Implements the non-scope justified defeater path tested for `defeat_reason_type` metadata.
- [function] `reasons/api.py:retract_node` — Drives the truth-maintenance path that turns a scope belief `OUT`, cascades to the defeater, and restores the target.
- [file] `reasons/derive.py` — Likely contains truth-value propagation and cascade logic exercised by dependent-node and retraction tests.
- [file] `tests/test_defeat_justification.py` — Covers the adjacent non-scope defeat behavior that this file only touches for reason-type metadata.

## Beliefs

- `defeat-with-scope-creates-scope-beliefs-as-in-premises` — Each scope finding produces one `IN` premise node with no justifications and metadata linking it to the antecedent and defeater.
- `defeater-requires-scope-beliefs-as-SL-antecedents` — The generated defeater is an `IN` derived node whose single `SL` justification uses the returned `scope_belief_ids` as antecedents.
- `retracting-scope-belief-restores-target` — In the tested single-justification scenario, retracting a scope belief makes both the scope belief and defeater `OUT` and restores the target to `IN`.
- `defeat-with-scope-cascades-to-dependents` — Defeating a node through scope beliefs propagates `OUT` to nodes whose justifications depend on the defeated node.
- `defeat-with-scope-validates-inputs-with-specific-exceptions` — Missing targets raise `KeyError`, no-justification targets and empty findings raise `ValueError`, and invalid indices raise `IndexError`.
