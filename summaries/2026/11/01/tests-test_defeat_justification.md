---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_defeat_justification.py

# `tests/test_defeat_justification.py`

## 1. Purpose

This file is the integration test suite for the **graph-native justification defeater** mechanism in the `ftl-reasons` project. The project implements a defeasible (non-monotonic) belief network where nodes have truth values (`IN`/`OUT`) and are supported by *justifications* (sets of source premises). This test file verifies the `api.defeat_justification` API — the operation that introduces a defeater node targeting a specific justification index of a belief, thereby potentially flipping that belief from `IN` to `OUT`.

It owns the responsibility of ensuring:
- The defeat mechanism correctly modifies the belief's truth state.
- Defeaters are graph-native nodes (first-class citizens in the network), not just annotations.
- Cascading, restoration, multi-justification, error, and export semantics all hold.

## 2. Key Components

| Function | Contract |
|----------|----------|
| `test_defeat_justification_basic` | Given a belief with one justification, calling `defeat_justification` flips it `IN → OUT`, creates a defeater node with deterministic ID `{type}-{node_id}-j{index}`, and records metadata (`defeater_type`, `defeats_node`, `defeats_justification`). |
| `test_defeat_justification_added_to_outlist` | The defeater's ID must appear in the target justification's `outlist` array — this is the graph-level link that makes defeaters "native" to the structure. |
| `test_defeat_restoration_on_retract` | Calling `api.retract_node(defeater_id)` must restore the belief to `IN` and report it in `changed`. This is the undo semantics. |
| `test_defeat_cascade_to_dependents` | If `b2` depends on `b1`, defeating `b1`'s justification must also flip `b2` to `OUT` (transitive cascade). Both appear in the `changed` set. |
| `test_defeat_multiple_justifications` | A belief with N justifications stays `IN` until **all** justifications are defeated. Defeating index 0 of a 2-justification belief does not flip it. |
| `test_defeat_custom_id` | The `defeater_id` parameter overrides the auto-generated ID; the custom ID is retrievable via `show_node`. |
| `test_defeat_errors` | Three error paths: missing node → `KeyError`; node with no justifications → `ValueError`; out-of-range index → `IndexError`. |
| `test_defeater_types` | Four recognized types: `invalid-inference`, `over-generalizes`, `duplicate-of`, `superseded-by`. Each is stored in `metadata["defeater_type"]`. |
| `test_defeater_in_export` | `api.export_network` must serialize defeaters as nodes and preserve the outlist linkage. |
| `test_premise_defeat` | Premises (nodes without `sl`/justifications) must raise `ValueError` on defeat — they are axiomatic and cannot be defeated at the justification level. |

The primary API under test is `api.defeat_justification(node_id, justification_index, reason, *, defeater_type=None, defeater_id=None, db_path=None)`.

## 3. Patterns

- **Fresh-DB-per-test**: Every test creates its own SQLite database via `tmp_path / "test.db"` and `api.init_db(...)`. No shared state, no inter-test dependencies. This is the standard pytest `tmp_path` idiom for isolation.
- **Arrange → Act → Assert**: Each test follows the classic pattern: build a minimal belief graph (arrange), call `defeat_justification` (act), then assert on truth values, metadata, outlists, and return-dict fields (assert).
- **Graph-native modeling of defeaters**: Defeaters are not side-tables or flags — they are full nodes in the same graph. This is verified by `test_defeater_in_export` and `test_defeat_restoration_on_retract` (you can `retract_node` them like any other node).
- **Deterministic ID generation**: The default defeater ID follows `{defeater_type}-{target_node}-j{index}` (e.g., `over-generalizes-belief-j0`). One test (`test_defeat_justification_basic`) asserts this exact string.
- **Cascade verification**: The test for cascading uses a linear chain (`p → b1 → b2`) rather than a diamond, keeping the causal path unambiguous.
- **Error-path testing via `pytest.raises`**: All three error conditions are tested in a single function (`test_defeat_errors`) with specific exception types and regex-matched messages.

## 4. Dependencies

**This file imports:**
- `pytest` — test runner, `tmp_path` fixture, `pytest.raises`
- `reasons.api` — the public API module (`init_db`, `add_node`, `add_justification`, `defeat_justification`, `show_node`, `retract_node`, `export_network`)

**What depends on this file:**
- Nothing imports this test file. It is a leaf test module. It is discovered by pytest via the `tests/` package (`tests/__init__.py`).
- It implicitly depends on `tests/conftest.py` for any shared fixtures (though this file only uses `tmp_path`, a built-in pytest fixture).

**Sister test files that exercise adjacent functionality:**
- `test_defeat_with_scope.py` — defeaters with scoped/limited blast radius
- `test_classify_defeaters.py` — classification logic for defeater types
- `test_migrate_defeaters.py` — migration of legacy defeater representations
- `test_remove_justification.py` — removal (vs. defeasibility) of justifications

## 5. Flow

Each test follows this execution sequence:

```
1. tmp_path provides an isolated directory
2. api.init_db(str(db))          → creates a fresh SQLite schema
3. api.add_node(...)             → inserts premise/belief nodes (and justifications via `sl`)
4. [optional] api.add_justification(...) → adds a second justification
5. api.show_node(...)            → asserts pre-condition (belief is IN)
6. api.defeat_justification(...) → the operation under test
   └─ internally:
      a. validates node exists, has justifications, index is in range
      b. creates a new defeater node in the graph
      c. appends defeater_id to the target justification's outlist
      d. recomputes truth values (if all justifications now have outlist entries → OUT)
      e. cascades to dependent nodes (BFS/DFS through `sl` edges)
      f. returns {node_id, justification_index, defeater_type, defeater_id, changed}
7. api.show_node(...)            → asserts post-condition (truth value, metadata, outlist)
8. [optional] api.retract_node(defeater_id) → tests undo path
9. [optional] api.export_network() → tests serialization fidelity
```

The key data transformation in step 6d: a justification's `outlist` is the set of defeater IDs that block it. A node's truth value is `IN` iff **at least one** of its justifications has an empty (or non-blocking) outlist. This is the core non-monotonic rule.

## 6. Invariants

The tests collectively enforce these invariants:

1. **Last-justification rule**: A node is `OUT` if and only if *every* justification in its list has been defeated (i.e., each has a defeater in its outlist). Defeating one of two justifications does NOT flip the node.
2. **Cascade completeness**: If node A is `IN` only because of node B, and B goes `OUT`, then A must also go `OUT`. The `changed` set in the return value must include all transitively affected nodes.
3. **Defeater-as-node**: A defeater is a first-class graph node with `truth_value == "IN"` upon creation, retrievable by ID, exportable, and retractable.
4. **Reversibility**: `retract_node(defeater_id)` must restore the original truth state of all previously-affected nodes.
5. **Axiomatic premises are un-defeatable at the justification level**: A node with no `sl` (no justifications) cannot be targeted by `defeat_justification`.
6. **Deterministic default ID**: When `defeater_id` is not provided, the system generates `{defeater_type}-{target_node}-j{index}`.
7. **Export fidelity**: The `export_network` output must contain defeater nodes with correct metadata and the outlist linkage in the target justification.

## 7. Error Handling

Three distinct error paths are tested in `test_defeat_errors`:

| Condition | Exception | Message match |
|-----------|-----------|---------------|
| Target node does not exist | `KeyError` | `"not found"` |
| Node exists but has zero justifications (e.g., a premise) | `ValueError` | `"no justifications"` |
| `justification_index` exceeds the number of justifications | `IndexError` | `"out of range"` |

The tests use `pytest.raises(ExceptionType, match=regex)` to pin both the exception type and a substring of the message. This means the production code must:
- Raise (not return an error dict) for these conditions.
- Use the specific exception types listed (not a generic `Exception`).
- Include the matched substrings in the message.

There is no test for *concurrent* defeaters, *self-referential* defeaters, or defeaters targeting already-OUT justifications — those edge cases may be handled elsewhere or may be known gaps.

## Topics to Explore

- [function] `reasons/api.py:defeat_justification` — The production implementation this file tests; understanding its internal cascade logic, outlist mutation, and ID generation is essential before modifying the tests.
- [file] `tests/test_defeat_with_scope.py` — Tests defeaters with limited scope; understanding how "scoped" defeaters differ from the full-graph defeaters tested here clarifies the boundary of this file's coverage.
- [file] `reasons/verify.py` — Likely contains the truth-value re-evaluation logic (the IN/OUT propagation algorithm) that `defeat_justification` triggers; the cascade tests depend on its correctness.
- [file] `tests/test_migrate_defeaters.py` — Tests migration of legacy/flat defeater representations to the graph-native model tested here; important context for why defeaters are nodes and not annotations.
- [general] `non-monotonic-reasoning-outlist-semantics` — The broader theoretical framework: how the outlist-on-justification pattern implements Reiter's default logic / Toulmin argumentation, and how it differs from simple retraction.

## Beliefs

- `defeater-is-a-graph-node` — A defeater created by `defeat_justification` is a first-class node in the belief graph (has `truth_value`, is exportable, is retractable), not a side-table annotation on the justification.
- `defeat-justification-causes-cascade` — Defeating the last valid justification of a belief must transitively flip all downstream dependents to `OUT`, and all affected node IDs must appear in the returned `changed` set.
- `outlist-blocks-justification` — A justification is considered "blocked" if and only if its `outlist` array contains at least one defeater ID; a node is `IN` iff at least one justification is unblocked.
- `retract-defeater-restores-belief` — Calling `api.retract_node` on a defeater must restore the `IN` state of the previously-defeated belief and all cascaded dependents, making defeasibility fully reversible.
- `premise-has-no-justifications` — A node added without an `sl` (source list) parameter has zero justifications and will always raise `ValueError` when targeted by `defeat_justification`, enforcing that axiomatic premises cannot be defeated at the justification level.
