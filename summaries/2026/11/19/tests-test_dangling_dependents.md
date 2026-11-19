---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_dangling_dependents.py

# `tests/test_dangling_dependents.py`

## Purpose

This is a regression test module for issue **#22**: the belief-network propagation logic must tolerate **dangling dependent references**.

A node keeps a `dependents` set of node IDs that depend on it. In normal operation, every ID in `dependents` should correspond to a real node in `Network.nodes`. This test intentionally corrupts that invariant by inserting fake IDs such as `"ghost"`, `"nonexistent"`, `"ghost1"`, etc., into `node.dependents`.

The file owns the behavioral contract that when `_propagate()` encounters a dependent ID that is not present in `net.nodes`, it must:

1. not raise `KeyError`;
2. emit a warning event into `net.log`;
3. continue propagating all valid dependents;
4. keep dangling IDs out of the returned `changed` list;
5. not poison future behavior if that ID is later added as a real node.

It covers propagation paths through `retract()`, `assert_node()`, `challenge()`, and `add_justification()`.

---

## Key Components

### Imports

- `from reasons import Justification, Node`
  - `Justification` is used to create simple support-link justifications, e.g. `Justification(type="SL", antecedents=["a"])`.
  - `Node` is imported but not used anywhere in this file. It is likely a leftover import.
- `from reasons.network import Network`
  - Provides the main belief-network object under test.
  - The repo also contains a parallel `reasons_lib` package, but this test targets `reasons`.

### `TestDanglingDependentNoCrash`

Validates the basic no-crash guarantee.

- `test_retract_with_single_dangling_dependent`
  - Adds node `"a"`, injects `"nonexistent"` into `net.nodes["a"].dependents`, then calls `net.retract("a")`.
  - A `KeyError` or other propagation crash fails the test.

- `test_assert_with_single_dangling_dependent`
  - Retracts `"a"`, injects a ghost dependent, then calls `net.assert_node("a")`.
  - Verifies the assertion path through `_propagate()` also tolerates dangling refs.

- `test_multiple_dangling_refs_on_one_node`
  - Injects `{"ghost1", "ghost2", "ghost3"}` into one node’s `dependents`.
  - Verifies multiple bad IDs do not crash retraction.

- `test_dangling_ref_alongside_valid_dependent`
  - Builds `"a"` as a premise and `"b"` as an SL-justified dependent of `"a"`.
  - Injects `"ghost"` into `"a".dependents`.
  - Retracting `"a"` must still retract `"b"`: `net.nodes["b"].truth_value == "OUT"`.

### `TestDanglingDependentWarningLogs`

Validates the observability contract for warnings.

The tests assume warning records are appended to `net.log` as dictionaries with at least:

- `"action"`: must be `"warn"`
- `"target"`: the dangling/missing node ID
- `"value"`: human-readable warning text
- `"timestamp"`: present on warning entries

Tests include:

- `test_single_warning_logged`
  - Exactly one warning for one ghost dependent.
  - `warnings[0]["target"] == "ghost"`.
  - `"dangling"` appears in the warning value.

- `test_warning_includes_parent_node_id`
  - The warning value must identify the node whose `dependents` set contained the bad ID.
  - With node `"a"`, `"a" in warnings[0]["value"]` must be true.

- `test_multiple_dangling_produce_multiple_warnings`
  - Two ghosts on one node produce two warnings.
  - Targets are exactly `{"ghost1", "ghost2"}`.

- `test_warning_on_assert_path`
  - Clears the log after setup, then calls `assert_node("a")`.
  - Verifies warnings are emitted on the assertion propagation path too.

- `test_warning_has_timestamp`
  - Verifies warning records include `"timestamp"`.

### `TestDanglingDependentPropagationContinues`

Validates that dangling references do not stop or corrupt normal cascade behavior.

- `test_valid_dependent_still_retracted`
  - `"b"` depends on `"a"`.
  - Adds a ghost on `"a"`.
  - `retract("a")` must still put `"b"` in `changed` and make `"b"` `"OUT"`.

- `test_valid_dependent_still_restored`
  - `"a"` is retracted, making `"b"` `"OUT"`.
  - Adds a ghost on `"a"`.
  - `assert_node("a")` must still put `"b"` in `changed` and restore `"b"` to `"IN"`.

- `test_chain_with_dangling_in_middle`
  - Chain: `a -> b -> c`.
  - `"b"` has a ghost dependent.
  - Retracting `"a"` must cascade through `"b"` to `"c"`.
  - Asserts both `"b"` and `"c"` are in `changed`, and `"c"` is `"OUT"`.

- `test_diamond_with_dangling`
  - Diamond: `a -> b`, `a -> c`, and `b,c -> d`.
  - Ghost added to `"a".dependents`.
  - Retracting `"a"` must still make `"d"` `"OUT"`.
  - Asserts `"ghost" not in changed`.

### `TestDanglingDependentEdgeCases`

Covers less common paths and interactions.

- `test_same_dangling_in_multiple_nodes`
  - Same ghost ID is added to `"a".dependents` and `"b".dependents`.
  - Retracting `"a"` must emit at least one warning targeting `"ghost"`.
  - The assertion uses `>= 1`, so it tolerates one warning per encounter or deduplication.

- `test_dangling_not_in_changed_list`
  - A ghost dependent must not appear in the returned `changed` collection.

- `test_challenge_with_dangling`
  - Calls `net.challenge("a", "test challenge")`.
  - Verifies the `challenge()` path also routes through tolerant propagation and logs a ghost warning.

- `test_add_justification_with_dangling`
  - Adds a ghost dependent to `"b"`.
  - Adds a justification to `"b"` and retracts `"a"`.
  - Verifies at least one warning for `"ghost"` is produced through the `add_justification()`/retraction flow.

- `test_dangling_does_not_enter_visited`
  - Adds `"ghost"` as a dangling dependent of `"a"`.
  - Retracts `"a"`.
  - Then adds `"ghost"` as a real node justified by `"a"`.
  - Since `"a"` is `"OUT"`, `"ghost"` should be `"OUT"`.
  - Asserting `"a"` should restore `"ghost"` to `"IN"`.
  - This encodes the subtle invariant that missing IDs should not be permanently recorded in propagation’s visited set in a way that prevents them from being processed after they become real nodes.

---

## Patterns

- **Class-based pytest organization**
  - Tests are grouped by concern: no-crash behavior, warning logs, propagation continuation, edge cases.

- **Arrange-Act-Assert**
  - Each test creates a fresh `Network`, lays out nodes, injects a dangling dependent, invokes a mutation, then asserts on truth values, `changed`, or log entries.

- **White-box corruption for edge-case forcing**
  - The tests reach directly into `net.nodes["a"].dependents` and mutate the set.
  - This is intentional: there is no public API for creating dangling references, so the tests simulate corrupted or stale state.

- **Log-event assertions**
  - Warnings are not checked via Python’s `warnings` module or `caplog`.
  - Instead, the tests inspect `net.log` entries and filter on `entry["action"] == "warn"`.

- **Black-box behavior checks around internal traversal**
  - Most assertions use public APIs and observable results: `retract()`, `assert_node()`, `changed`, `truth_value`, and `log`.

- **Set-based dependent storage**
  - Because `dependents` is a set, iteration order is not guaranteed. Tests avoid order-dependent assertions except where only one warning exists.

---

## Dependencies

### This file depends on

- `reasons.Justification`
  - Used to create support-link justifications.
- `reasons.network.Network`
  - The system under test.
- Public or semi-public `Network` surface:
  - `add_node(node_id, text, justifications=[...])`
  - `retract(node_id) -> changed`
  - `assert_node(node_id) -> changed`
  - `challenge(node_id, reason)`
  - `add_justification(node_id, justification)`
  - `.nodes` mapping
  - node attributes:
    - `.dependents`
    - `.truth_value`
  - `.log` list of event dictionaries

### This file is depended on by

- The pytest test suite / CI discovery.
- It is not imported by application code.
- It acts as a regression guard for `reasons.network.Network._propagate()` and related mutation paths.

Given the repo has both `reasons/` and `reasons_lib/`, this test specifically validates the `reasons` package. If the project expects parity between both packages, this test does not directly guard `reasons_lib`.

---

## Flow

A typical execution pattern is:

1. Create a fresh `Network`.
2. Add real nodes, sometimes with `Justification(type="SL", antecedents=[...])`.
3. Inject one or more fake dependent IDs directly:
   ```python
   net.nodes["a"].dependents.add("ghost")
   ```
4. Trigger a mutation that invokes `_propagate()`:
   - `net.retract("a")`
   - `net.assert_node("a")`
   - `net.challenge("a", "...")`
   - `net.add_justification(...)` followed by retraction
5. During propagation, `_propagate()` iterates dependent IDs. For a missing ID:
   - it must skip node lookup/update;
   - it must append a warning event to `net.log`;
   - it must continue processing remaining dependents.
6. The test asserts:
   - no exception was raised;
   - valid nodes changed truth values correctly;
   - `changed` contains valid affected node IDs;
   - `changed` excludes dangling IDs;
   - warning records have the expected action, target, text, and timestamp.

For example, in the diamond test:

```text
a -> b
a -> c
b, c -> d
```

Retracting `"a"` marks `"a"` OUT, then propagates to `"b"` and `"c"`, then to `"d"`. The ghost dependent on `"a"` is encountered and skipped, but it must not interrupt the actual cascade. Final assertion:

```python
assert net.nodes["d"].truth_value == "OUT"
assert "ghost" not in changed
```

---

## Invariants

This test file enforces these rules:

- A dangling dependent ID must never cause `KeyError` or any crash in propagation.
- Dangling IDs must be skipped, not treated as real nodes.
- A warning must be logged for each encountered dangling dependent, with:
  - `action == "warn"`
  - `target` equal to the missing ID
  - warning text containing `"dangling"`
  - warning text containing the parent node ID
  - a timestamp
- Valid dependents must still be processed even if ghost entries exist in the same `dependents` set.
- Truth values must still cascade correctly through valid chains and diamonds despite dangling references.
- Dangling IDs must not appear in the returned `changed` collection.
- A previously dangling ID must not be permanently excluded from future propagation after it is added as a real node.
- The returned `changed` collection should contain real affected nodes, e.g. `"b"` when `"b"` is a valid dependent of a retracted node.

---

## Error Handling

- The tests intentionally do **not** use `pytest.raises`.
- The expected behavior is that the `Network` absorbs the invalid dependent reference as a warning event rather than raising.
- If `_propagate()` raises `KeyError`, the test fails naturally.
- Warning reporting is represented as `net.log` events, not as Python exceptions or the `warnings` module.
- Assertions fail if:
  - warning count is wrong;
  - `target` is wrong;
  - warning text does not contain expected context;
  - valid nodes fail to change truth value;
  - dangling IDs leak into `changed`.

---

## Topics to Explore

- [file] `reasons/network.py` — Contains the `Network` implementation and the `_propagate()` logic this test guards.
- [function] `reasons/network.py:_propagate` — Core traversal where dangling dependent IDs must be skipped, logged, and excluded from `changed`.
- [file] `tests/test_dependents_integrity.py` — Related tests around reverse dependency integrity and likely valid propagation behavior.
- [general] `dangling-dependents-issue-22` — Regression history and intended semantics for stale/dangling dependent references.
- [function] `reasons/network.py:retract` — Mutation entry point that triggers propagation and returns the `changed` collection asserted here.

## Beliefs

- `dangling-dependents-do-not-crash-propagation` — `Network._propagate()` must skip missing dependent node IDs instead of raising `KeyError`.
- `dangling-dependent-produces-warning-log` — Encountering a dangling dependent emits a `net.log` entry with `action == "warn"` and `target` equal to the missing ID.
- `warning-value-identifies-parent-and-dangling` — Warning text contains both `"dangling"` and the parent node ID whose `dependents` set held the bad reference.
- `propagation-continues-past-dangling-references` — Valid dependents still update truth values and are included in `changed` even when ghost dependents are present.
- `dangling-ids-not-in-changed-or-visited` — Dangling IDs are excluded from `changed` and must not prevent a later real node with the same ID from participating in propagation.
