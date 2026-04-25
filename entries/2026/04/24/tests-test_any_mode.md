# File: tests/test_any_mode.py

**Date:** 2026-04-24
**Time:** 16:54



# `tests/test_any_mode.py` — Explanation

## Purpose

This file is the test suite for three related features that were implemented together (issues #19, #20, #21). All three address a usability problem with conjunctive justifications: when a node depends on many premises via a single SL justification, retracting *any one* premise kills the node — which is often too aggressive. The three features are:

1. **`--any` mode (#20)**: An alternative to conjunctive SL. Instead of creating one justification requiring *all* premises, `any_mode=True` expands `sl="a,b,c"` into three separate single-premise justifications. The node stays IN as long as *any* premise remains IN (disjunctive semantics via the existing multiple-justification mechanism).

2. **Premise count warning (#19)**: `add_node` and `add_justification` return a `premise_count` field so the CLI can warn users when they create fragile conjunctive justifications with 3+ premises.

3. **Restoration hints (#21)**: When a retraction cascade knocks out a multi-premise node, the return value includes hints showing which premises survived — telling the user what they'd need to re-justify to restore the node.

## Key Components

### `TestAnyModeAdd`

Tests `any_mode=True` on `api.add_node`. The core contract:

- `any_mode=True` with `sl="a,b,c"` produces **3 justifications**, each with **1 antecedent** (not 1 justification with 3 antecedents).
- The node stays IN when any single premise is retracted (`test_any_survives_single_retraction`).
- The node goes OUT only when **all** premises are retracted (`test_any_goes_out_when_all_retracted`).
- With a single premise, `any_mode` is a no-op — no expansion needed (`test_any_with_single_premise_no_expansion`).
- Outlists are preserved: each expanded justification inherits the `unless` clause (`test_any_preserves_outlist`).
- Without `any_mode`, the default conjunctive behavior is unchanged (`test_without_any_single_justification`).

### `TestAnyModeAddJustification`

Tests `any_mode=True` on `api.add_justification` (adding justifications to an existing node). Verifies that expansion works post-creation and can restore an OUT node.

### `TestPremiseCountWarning`

Tests the `premise_count` field in return values:

- Conjunctive `sl="a,b,c"` → `premise_count == 3`
- `any_mode=True` with same → `premise_count == 1` (each justification has 1 premise)
- A premise node (no SL) → `premise_count == 0`

This data enables the CLI to print a warning like "Warning: 3 premises — consider --any" without the API layer making UI decisions.

### `TestRestorationHints`

Tests the `restoration_hints` list returned by `api.retract_node`. Hints are produced when:

- A multi-premise SL node goes OUT due to cascade, AND
- At least one of its premises is still IN (surviving).

Hints are **not** produced when:
- All premises went OUT (nothing to suggest).
- The SL has only 1 premise (nothing partial to report).
- The node was the one directly retracted (user already knows).
- The node was created with `--any` and stays IN (no cascade occurred).

Each hint contains `node_id`, `all_premises`, and `surviving_premises`.

## Patterns

- **Fixture-per-test isolation**: The `db` fixture creates a fresh SQLite database in `tmp_path` for each test. No shared state between tests.
- **API-level testing**: Tests go through `reasons_lib.api` — the functional Python API layer — not through the CLI or directly through `Network`. This is the right boundary: it tests business logic without CLI parsing noise, but includes the full storage round-trip.
- **Return-value-driven assertions**: The tests assert on dictionary return values from API calls. The API layer is designed to return structured data (dicts with keys like `truth_value`, `justifications`, `premise_count`, `restoration_hints`) rather than raising exceptions or printing output.
- **Issue-tagged test classes**: Each class references the GitHub issue it validates (#19, #20, #21). The module docstring lists all three.

## Dependencies

**Imports**: `pytest` (test framework), `reasons_lib.api` (the functional API being tested).

**Imported by**: Nothing — this is a leaf test module. It's discovered and executed by pytest.

**Transitive dependencies**: `api.py` calls into `network.py` (propagation engine) and `storage.py` (SQLite persistence). These tests exercise the full stack from API → Network → SQLite → filesystem.

## Flow

Each test follows the same pattern:

1. **Setup**: Create premise nodes (`a`, `b`, `c`) via `api.add_node`.
2. **Act**: Create a derived node with `sl=` (and optionally `any_mode=True`, `unless=`), or retract a premise.
3. **Assert**: Inspect the resulting node state via `api.show_node` or check the return value of the mutating call.

The `any_mode` expansion happens inside `api.add_node` / `api.add_justification`: the API translates `sl="a,b,c", any_mode=True` into three separate `add_justification` calls with `sl="a"`, `sl="b"`, `sl="c"` respectively. The Network's existing multiple-justification semantics (a node is IN if ANY justification is valid) then naturally provides disjunctive behavior.

## Invariants

1. **Any-mode expansion**: `any_mode=True` with N premises produces exactly N justifications, each with exactly 1 antecedent.
2. **Outlist inheritance**: Every expanded justification gets the same outlist as the original would have.
3. **Premise count accuracy**: `premise_count` reflects the max antecedent count across the node's justifications, not the total premise count.
4. **Hint conditions**: Restoration hints appear only for cascade victims (not the directly retracted node) that have multi-premise SLs with at least one surviving premise.
5. **No false alarms**: Any-mode nodes that stay IN produce no restoration hints.

## Error Handling

These tests don't exercise error paths — they test the happy path of three tightly coupled features. The API layer returns dicts rather than raising, so the tests assert on dict contents. If the API fails, it would raise an unhandled exception that pytest catches as a test failure.

---

## Topics to Explore

- [function] `reasons_lib/api.py:add_node` — Where `any_mode` expansion actually happens; see how it loops over premises and calls `add_justification` per-premise
- [function] `reasons_lib/api.py:retract_node` — Where `restoration_hints` are computed during the cascade; the logic that filters for multi-premise SLs with surviving premises
- [file] `reasons_lib/network.py` — The propagation engine that makes multiple justifications work as disjunction (a node is IN if ANY justification is valid)
- [file] `reasons_lib/cli.py` — Where `premise_count` gets turned into a user-visible warning and `restoration_hints` get formatted for terminal output
- [general] `conjunctive-vs-disjunctive-justifications` — Understanding why the default SL is conjunctive (all must be IN) and how `any_mode` achieves disjunction through structural expansion rather than a new justification type

## Beliefs

- `any-mode-is-structural-expansion` — `any_mode` does not introduce a new justification type; it reuses existing multi-justification semantics by expanding one N-premise SL into N single-premise SLs
- `premise-count-is-per-justification-max` — `premise_count` returns the count of antecedents in the largest justification, not the total across all justifications (any_mode with 3 premises returns 1, not 3)
- `restoration-hints-require-surviving-premises` — A retraction cascade only produces a restoration hint for a node if at least one of its SL premises is still IN after the cascade
- `any-mode-outlist-preserved` — When `any_mode` expands `sl="a,b" unless="enemy"`, each resulting single-premise justification includes `"enemy"` in its outlist
- `hints-exclude-directly-retracted-node` — The node passed to `retract_node` never appears in the `restoration_hints` list, only cascade victims do

