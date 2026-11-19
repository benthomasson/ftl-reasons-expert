---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_deduplication.py

# `tests/test_deduplication.py`

## Purpose

This file is an integration test suite for the public API behavior around deduplication and supersession metadata in the `reasons` package. Specifically, it validates that:

- a node can be marked as a duplicate of a canonical node;
- a node can be marked as superseded by a newer node;
- both operations set the affected node to `OUT`;
- both operations attach audit metadata (`duplicate_of` / `superseded_by` and `retract_reason`);
- dependent nodes are cascaded to `OUT`;
- missing-node and self-reference errors are handled correctly;
- the metadata persists and is visible through `show_node` and `export_network`.

The file owns the behavioral contract for `api.mark_duplicate` and `api.mark_superseded` at the API level, using a real temporary SQLite database rather than mocks.

---

## Key Components

### Test fixtures / setup

Every test uses pytest’s `tmp_path` fixture:

```python
db = tmp_path / "test.db"
api.init_db(str(db))
```

This gives each test an isolated database file. All API calls pass `db_path=str(db)`, so the tests exercise the same database path consistently.

### `test_mark_duplicate`

Tests the basic duplicate workflow:

- create `canonical-node` and `duplicate-node`;
- call:

```python
api.mark_duplicate("duplicate-node", "canonical-node", db_path=str(db))
```

Asserts the result contains:

```python
result["source_id"] == "duplicate-node"
result["canonical_id"] == "canonical-node"
"duplicate-node" in result["changed"]
```

Then verifies through `api.show_node` that the source node is now:

- `truth_value == "OUT"`
- `metadata["duplicate_of"] == "canonical-node"`
- `metadata["retract_reason"]` contains `"Duplicate of canonical-node"`

### `test_mark_superseded`

Tests the analogous supersession workflow:

```python
api.mark_superseded("old-belief", "new-belief", db_path=str(db))
```

Asserts:

```python
result["old_id"] == "old-belief"
result["new_id"] == "new-belief"
"old-belief" in result["changed"]
```

Then verifies:

- `old-belief` is `OUT`;
- `metadata["superseded_by"] == "new-belief"`;
- `metadata["retract_reason"]` contains `"Superseded by new-belief"`.

### Cascade tests

#### `test_mark_duplicate_with_cascade`

Creates a dependent node using:

```python
api.add_node("dependent", "Depends on duplicate", sl="duplicate", db_path=str(db))
```

Before marking, it verifies the dependent is `IN`. After:

```python
api.mark_duplicate("duplicate", "canonical", db_path=str(db))
```

it asserts:

- `"dependent" in result["changed"]`
- the dependent’s `truth_value` is now `OUT`.

#### `test_mark_superseded_with_cascade`

Same pattern, but the dependent points at `"old"` through `sl="old"` and the test calls `api.mark_superseded("old", "new", ...)`.

This confirms truth-maintenance cascades apply to both duplicate and superseded transitions.

### Error tests

#### `test_mark_duplicate_errors`

Verifies that missing IDs raise `KeyError` with a message matching `"not found"`:

```python
with pytest.raises(KeyError, match="not found"):
    api.mark_duplicate("missing", "exists", db_path=str(db))

with pytest.raises(KeyError, match="not found"):
    api.mark_duplicate("exists", "missing", db_path=str(db))
```

So both the source and canonical targets must exist before marking.

#### `test_mark_superseded_errors`

Same contract for `mark_superseded`: both old and new IDs must exist, otherwise `KeyError` is raised.

### Metadata visibility tests

#### `test_metadata_in_export`

Marks one duplicate and one superseded node, then calls:

```python
data = api.export_network(db_path=str(db))
```

It verifies that exported nodes include the same persisted metadata:

```python
dup = data["nodes"]["duplicate"]
assert dup["metadata"]["duplicate_of"] == "canonical"
assert dup["truth_value"] == "OUT"

old = data["nodes"]["old"]
assert old["metadata"]["superseded_by"] == "new"
assert old["truth_value"] == "OUT"
```

This ensures the relationship metadata is part of the serialized network representation, not just transient API output.

#### `test_metadata_in_show_output`

Verifies that `api.show_node` exposes the duplicate metadata and retract reason directly:

```python
node["metadata"]["duplicate_of"] == "canonical"
"retract_reason" in node["metadata"]
```

### Self-reference tests

#### `test_mark_duplicate_self_reference`

Requires:

```python
with pytest.raises(ValueError, match="cannot be marked as a duplicate of itself"):
    api.mark_duplicate("node-a", "node-a", db_path=str(db))
```

After the exception, it verifies the node remains `IN`, which implies validation happens before mutation.

#### `test_mark_superseded_self_reference`

Same pattern with:

```python
with pytest.raises(ValueError, match="cannot be marked as superseded by itself"):
    api.mark_superseded("node-a", "node-a", db_path=str(db))
```

Again, the node remains `IN`.

### Already-`OUT` node tests

#### `test_mark_duplicate_already_out`

Creates a node, retracts it first:

```python
api.retract_node("duplicate", reason="Initially retracted", db_path=str(db))
```

Then marks it as duplicate. The test asserts:

- the operation succeeds;
- the node remains `OUT`;
- `metadata["duplicate_of"] == "canonical"`;
- `metadata["retract_reason"] == "Duplicate of canonical"`.

This is important: marking as duplicate overwrites the previous retract reason with the deduplication reason.

#### `test_mark_superseded_already_out`

Same behavior for supersession. After:

```python
api.mark_superseded("old", "new", db_path=str(db))
```

the old node is still `OUT`, its metadata contains `superseded_by`, and its retract reason becomes exactly:

```python
"Superseded by new"
```

The previous `"Initially retracted"` reason is replaced.

---

## Patterns

### Arrange-Act-Assert

Each test follows a clear setup, mutation, and assertion sequence. The API state is observed through `show_node` or `export_network` rather than by inspecting database internals.

### Real SQLite integration tests

The tests do not mock storage. They use `tmp_path` and initialize a real database with `api.init_db`. This gives higher confidence that metadata persistence and truth-value updates work end-to-end.

### Public API facade testing

The test file imports only:

```python
from reasons import api
```

It exercises public functions such as:

- `api.init_db`
- `api.add_node`
- `api.mark_duplicate`
- `api.mark_superseded`
- `api.retract_node`
- `api.show_node`
- `api.export_network`

This means the tests are intentionally coupled to the public behavior of `reasons.api`, not to internal storage helpers.

### Structured mutation results

Both mutation APIs return dictionaries with named fields:

- `mark_duplicate` returns `source_id`, `canonical_id`, and `changed`;
- `mark_superseded` returns `old_id`, `new_id`, and `changed`.

The tests rely on these stable result keys and on `changed` containing both the directly mutated node and any cascaded dependents.

### Deterministic retract reasons

The tests assert exact or substring retract reasons:

- `"Duplicate of canonical"`
- `"Superseded by new"`
- `"Superseded by new-belief"`

This establishes a deterministic audit trail format.

---

## Dependencies

### Imports used by this file

- `pytest` — test runner, `tmp_path` fixture, and `pytest.raises`.
- `reasons.api` — the public API facade being tested.

### Production modules under test

The behavior ultimately depends on whatever `reasons.api` delegates to. Based on the repository layout, likely implementation lives in one or more of:

- `reasons/api.py`
- `reasons/storage.py`
- possibly truth-maintenance / metadata logic elsewhere in `reasons/`

The test file itself is not imported by production code. It is collected and executed by pytest.

---

## Flow

A typical test flow is:

1. Create a temporary SQLite path:
   ```python
   db = tmp_path / "test.db"
   ```

2. Initialize the database:
   ```python
   api.init_db(str(db))
   ```

3. Create the necessary nodes with `api.add_node(..., db_path=str(db))`.

4. Perform the mutation:
   - `api.mark_duplicate(...)`
   - or `api.mark_superseded(...)`

5. Inspect the returned result dictionary for IDs and `changed` entries.

6. Re-read nodes using `api.show_node(...)` or `api.export_network(...)` and assert the persisted state.

For cascade tests, the flow adds one extra step: create a dependent node using `sl="<source-id>"`, verify it starts `IN`, perform the deduplication/supersession, then verify the dependent has been cascaded to `OUT`.

For error tests, the flow changes at step 4: the expected outcome is an exception rather than a returned result.

---

## Invariants

This test file enforces several important invariants:

1. **Referential integrity**  
   `mark_duplicate` requires both the source and canonical node to exist.  
   `mark_superseded` requires both the old and new node to exist.  
   Missing IDs raise `KeyError` with `"not found"`.

2. **No self-relations**  
   A node cannot be marked as a duplicate of itself or superseded by itself.  
   These cases raise `ValueError`.

3. **Validation before mutation**  
   Self-reference errors leave the node unchanged. The self-reference tests assert the node remains `IN`.

4. **Truth-value transition**  
   Marking a node as duplicate or superseded sets that node to `OUT`.

5. **Metadata persistence**  
   `mark_duplicate` writes `metadata["duplicate_of"]`.  
   `mark_superseded` writes `metadata["superseded_by"]`.  
   Both write a `metadata["retract_reason"]`.

6. **Deterministic audit reason**  
   The retract reason is derived from the operation:
   - `"Duplicate of <canonical-id>"`
   - `"Superseded by <new-id>"`

7. **Already-`OUT` nodes are supported**  
   Both operations can be applied to a node that was already retracted. In that case, the new deduplication/supersession reason overwrites the previous retract reason.

8. **Dependent cascade**  
   Dependents connected to the affected node via `sl` are transitioned to `OUT` and appear in the mutation result’s `changed` list.

9. **Export parity**  
   Metadata and truth values visible through `show_node` are also present in `export_network` output.

---

## Error Handling

The API is expected to raise exceptions rather than return error dictionaries for the cases tested here.

- **Missing node IDs** produce `KeyError` with a message matching `"not found"`.
- **Self-reference attempts** produce `ValueError` with exact message fragments:
  - `"cannot be marked as a duplicate of itself"`
  - `"cannot be marked as superseded by itself"`

The self-reference tests additionally verify that failure is atomic: the node is still `IN` after the exception. The already-`OUT` tests verify that deduplication and supersession are not treated as errors when the source node is already retracted.

---

## Topics to Explore

- [function] `reasons/api.py:mark_duplicate` — Implements duplicate marking, metadata update, and dependent cascade behavior tested here.
- [function] `reasons/api.py:mark_superseded` — Implements supersession logic and its retract-reason/metadata contract.
- [function] `reasons/api.py:retract_node` — Underlying retraction behavior that `mark_duplicate` and `mark_superseded` build on for already-`OUT` cases.
- [file] `reasons/storage.py` — Likely owns persistence of node metadata, truth values, and export serialization.
- [general] `truth-maintenance-cascade` — Investigate how `sl` dependents are discovered and forced to `OUT` after source retraction.

## Beliefs

- `mark-duplicate-requires-existing-source-and-canonical` — `api.mark_duplicate` raises `KeyError("not found")` when either ID is missing.
- `mark-superseded-requires-existing-old-and-new` — `api.mark_superseded` raises `KeyError("not found")` when either ID is missing.
- `self-deduplication-validation-prevents-mutation` — Self-reference calls raise `ValueError` and leave the node’s truth value unchanged.
- `deduplication-and-supersession-cascade-to-dependents` — Dependents created with `sl=<source-id>` are listed in `result["changed"]` and become `OUT`.
- `mark-operations-overwrite-existing-retract-reason` — Marking an already-retracted node replaces its prior `retract_reason` with the duplicate/supersession reason.
