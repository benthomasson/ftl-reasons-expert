---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_any_mode.py

# `tests/test_any_mode.py`

## Purpose

This file is the integration test suite for three interrelated features in the `ftl-reasons` belief-network library, each tracked by a GitHub issue number:

- **#20 — `--any` flag**: Allows a node's support list (SL) to be interpreted disjunctively rather than conjunctively. Instead of requiring *all* premises to be IN, *any one* suffices.
- **#19 — Premise-count warning**: Exposes `premise_count` in API return values so the CLI can warn users when they build a 3+ premise conjunctive SL (which is fragile — one retraction kills the whole justification).
- **#21 — Restoration hints**: When a retraction cascades and takes a multi-premise node OUT, the API returns "hints" telling the user which premises are still IN, so they know what to restore.

The file is the **behavioral contract** for these three features. It doesn't test internals; it tests the public `reasons.api` surface (`add_node`, `add_justification`, `retract_node`, `show_node`).

## Key Components

### Fixture

```python
@pytest.fixture
def db(tmp_path):
```
Creates a throwaway SQLite database per test via `api.init_db()`. Every test gets an isolated database — no cross-test contamination.

### `TestAnyModeAdd`

Tests the `any_mode=True` parameter on `api.add_node()`:

| Test | Contract verified |
|------|-------------------|
| `test_any_creates_separate_justifications` | 3 premises → 3 justifications, each with exactly 1 antecedent (disjunction expansion) |
| `test_any_survives_single_retraction` | Retract one premise → node stays IN (other justifications still hold) |
| `test_any_goes_out_when_all_retracted` | Retract *all* premises → node goes OUT |
| `test_any_with_single_premise_no_expansion` | 1 premise + `any_mode=True` → still 1 justification (no degenerate expansion) |
| `test_any_preserves_outlist` | `unless` clause propagates to *every* expanded justification's outlist |
| `test_without_any_single_justification` | Counterfactual: without `any_mode`, 3 premises → 1 justification with 3 antecedents (conjunctive) |

### `TestAnyModeAddJustification`

Tests `api.add_justification()` with `any_mode=True` on an already-existing node:

- `test_any_expands_add_justification` — Adding a 2-premise any-justification to a node that already has 1 SL yields 3 total justifications.
- `test_any_add_justification_restores_out_node` — An OUT node becomes IN after an any-justification is added (the new path provides support).

### `TestPremiseCountWarning`

Verifies the `premise_count` field in return dicts:

- Normal 3-premise SL → `premise_count == 3`
- `any_mode` 3-premise SL → `premise_count == 1` (each justification only has 1 premise)
- Premise-only node (no SL) → `premise_count == 0`

This is the data the CLI uses to emit the "consider using `--any`" warning.

### `TestRestorationHints`

Tests the `restoration_hints` list in the return of `api.retract_node()`:

- `test_hint_when_multi_premise_cascades` — Retracting one of 3 premises produces a hint with `all_premises`, `surviving_premises`, and the retracted node excluded from survivors.
- `test_no_hint_when_all_premises_out` — If cascading takes *all* premises OUT, no hint (nothing to restore).
- `test_no_hint_for_single_premise_sl` — Single-premise SL is all-or-nothing; no hint.
- `test_no_hint_for_directly_retracted_node` — The node you explicitly retracted doesn't self-hint.
- `test_hint_with_any_flag_no_false_alarm` — `--any` nodes don't produce hints because they never cascade (each justification is independent).
- `test_hint_includes_correct_surviving_premises` — Cascading (a→b→x) correctly excludes both `a` and `b` from survivors, leaving only `["c"]`.

## Patterns

1. **Behavioral / black-box testing**: Every test calls only `reasons.api` public functions. No internal state is inspected directly — assertions go through `show_node()` return dicts or API return values. This makes the tests robust to internal refactoring.

2. **Arrange-Act-Assert in explicit steps**: Each test builds a small graph (3–5 nodes), performs one API call (the "act"), then asserts on the resulting state. The arrangement is always: create premises → create conclusion → (optionally) retract/add → assert.

3. **Counterfactual pairing**: `test_any_creates_separate_justifications` is paired with `test_without_any_single_justification` to lock in the *contrast* between disjunctive and conjunctive SL. A reader seeing one test immediately understands the other's purpose.

4. **Issue-numbered docstrings**: Every class docstring references the GitHub issue (`#20`, `#19`, `#21`). This makes it trivially greppable when backporting or auditing feature coverage.

5. **`tmp_path` isolation**: The standard pytest `tmp_path` fixture ensures no shared state. The `db` fixture wraps it with `api.init_db()`, making the test fully hermetic.

## Dependencies

**Imports:**
- `pytest` — test runner, provides `fixture`, `tmp_path`
- `reasons.api` — the entire public API under test (`init_db`, `add_node`, `add_justification`, `retract_node`, `show_node`)

**Imported by (reverse dependencies):**
- This is a leaf test file. Nothing imports from it.
- It is collected by pytest via the `tests/` package (`tests/__init__.py` exists).
- `tests/conftest.py` may provide additional fixtures, though this file is self-contained.

**Sister tests that share the same API surface:**
- `tests/test_dangling_dependents.py`, `tests/test_defeat_justification.py`, `tests/test_backtracking.py` — all exercise the retraction cascade logic that `TestRestorationHints` depends on.

## Flow

```
test setup
  └─ fixture `db` → api.init_db(tmp_path) → returns db_path string

each test (using db):
  1. Arrange: api.add_node("a", "A", db_path=db)  × N
  2. Act:     api.add_node("x", "Conclusion", sl="a,b,c", any_mode=True, db_path=db)
              OR api.retract_node("a", db_path=db)
  3. Assert:  node = api.show_node("x", db_path=db)
              assert node["truth_value"] == "IN"
              assert len(node["justifications"]) == 3
              ...
```

For the `retract_node` tests specifically:
1. Build a DAG where `a` is a premise of `x` (and possibly of `b`, which is also a premise of `x`).
2. Call `api.retract_node("a")` — this triggers the cascade engine.
3. Assert on `result["restoration_hints"]` — a list of dicts with keys `node_id`, `all_premises`, `surviving_premises`.

The key data transformation is: **SL string `"a,b,c"` → internal justification records**. In `any_mode`, the API splits this into N records each with a single antecedent. In normal mode, it's 1 record with N antecedents.

## Invariants

1. **Disjunctive completeness**: If *any* single-premise justification of an `--any` node has all its antecedents IN, the node must be IN. (Enforced by `test_any_survives_single_retraction`.)

2. **Outlist propagation**: Every justification created by `any_mode` expansion must inherit the node's `unless` list. (Enforced by `test_any_preserves_outlist`.)

3. **`premise_count` reflects per-justification width, not total SL size**: For `any_mode`, it's always 1 (or 0 for a bare premise). For normal SL, it equals the number of premises. (Enforced by `TestPremiseCountWarning`.)

4. **Hints only for multi-premise conjunctive nodes**: A restoration hint appears only when (a) the node has a single justification with >1 antecedents, (b) at least one antecedent is still IN, and (c) the node was *not* the one directly retracted. (Enforced by `TestRestorationHints`.)

5. **Ordering in `surviving_premises`**: The list reflects the original SL order minus the OUT premises. `test_hint_includes_correct_surviving_premises` asserts `== ["c"]` (a list, implying order matters).

## Error Handling

This test file does **not** test error paths (no `pytest.raises` blocks). All tests exercise the happy path. Notable absences:

- No test for `add_node` with a non-existent premise ID (would the API raise or silently ignore?)
- No test for retracting a non-existent node
- No test for `any_mode=True` with an empty `sl` string

This is a gap: the tests verify *correct behavior* but not *failure semantics*. A maintainer adding validation logic to `api.add_node` would need to add new tests here.

The one implicit error contract is that `api.show_node()` returns a dict (never raises for a valid node ID) and that `api.retract_node()` returns a dict with a `restoration_hints` key (never missing). The tests assume these keys always exist, even when the list is empty (`assert result["restoration_hints"] == []`).

## Topics to Explore

- [file] `reasons/api.py` — The implementation of `add_node`, `add_justification`, and `retract_node` that these tests exercise; shows how `any_mode` splits the SL internally and how `restoration_hints` are computed during cascade.
- [file] `reasons/repair.py` — Likely the cascade engine that determines which nodes go OUT when a premise is retracted; the restoration-hint logic in `test_restoration_hints` depends on this module's traversal order.
- [file] `tests/test_dangling_dependents.py` — Tests the edge case where retracting a premise leaves downstream nodes with zero surviving justifications; complements the "all premises OUT" case in this file.
- [file] `reasons/cli.py` — Where `premise_count` is consumed to emit the "consider `--any`" warning to the user; understanding the CLI threshold (3+?) clarifies the purpose of issue #19.
- [general] `defeasible-reasoning-semantics` — The project implements a form of defeasible logic (IN/OUT truth values, outlists, justifications). Reading `reasons/verify.py` or `reasons/contradictions.py` will ground what "IN via a justification" formally means in this system.

## Beliefs

- `any-mode-expands-sl-to-per-premise-justifications` — `add_node(sl="a,b,c", any_mode=True)` produces 3 justification records each with exactly 1 antecedent, rather than 1 record with 3 antecedents.
- `any-mode-node-survives-single-premise-retraction` — An `--any` node with 3 premises remains IN after any one premise is retracted, because the other two justifications independently support it.
- `premise-count-reflects-justification-width-not-sl-length` — For `any_mode` nodes, `premise_count` is 1 (the width of each individual justification), not 3 (the total SL size); this is the signal the CLI uses to suppress the "too many premises" warning.
- `restoration-hints-require-surviving-premises` — A `restoration_hint` is emitted for a cascaded-OUT node only when at least one premise of its conjunctive SL is still IN; if all premises cascade OUT, no hint is produced.
- `outlist-propagates-to-all-expanded-justifications` — When `any_mode` splits an SL into N justifications, the `unless` (outlist) is copied to each one, ensuring defeat-by-exception applies uniformly regardless of which premise path supports the node.
