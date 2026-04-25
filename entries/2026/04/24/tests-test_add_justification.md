# File: tests/test_add_justification.py

**Date:** 2026-04-24
**Time:** 16:54

# `tests/test_add_justification.py` — Explanation

## Purpose

This file tests the `add_justification` feature: the ability to append additional justifications to nodes that already exist in the belief network. This is distinct from creating a node with justifications — it's about giving an existing node a new reason to be believed *after the fact*. The file validates this at two layers: the in-memory `Network` class and the persistence-backed `api` module.

This matters because multiple justifications are what make a TMS resilient. A node with one justification is fragile — retract its antecedent and it dies. A node with two justifications survives losing one. This file is the contract for that resilience.

## Key Components

### `TestNetworkAddJustification` (8 tests)

Tests `Network.add_justification()` directly — no database, no serialization. Each test constructs a small network by hand, calls `add_justification`, and asserts on truth values and structural properties.

Key tests and what they prove:

| Test | What it verifies |
|------|-----------------|
| `test_add_justification_to_derived_node` | Basic append: node keeps both justifications, stays IN, return dict has correct shape |
| `test_added_justification_keeps_node_in_after_retract` | **Core resilience**: node with two justifications survives when one antecedent is retracted |
| `test_add_justification_restores_out_node` | Adding a valid justification to an OUT node flips it to IN; `changed` list includes the node |
| `test_add_justification_cascades` | Restoration propagates: if `b` comes back IN, its dependent `d` also comes back IN |
| `test_add_justification_with_outlist` | Non-monotonic justification: outlist is respected. `b` stays OUT while `enemy` is IN, then goes IN when `enemy` is retracted |
| `test_add_justification_to_premise` | Premises (no justifications) can receive justifications, giving them a "backup route" |
| `test_add_justification_nonexistent_node` | Raises `KeyError` for unknown node IDs |
| `test_add_justification_registers_dependents` | Antecedent and outlist nodes get the target registered in their `dependents` set — critical for propagation to work |

### `TestAPIAddJustification` (8 tests)

Tests `api.add_justification()` — the persistence-backed functional API that wraps the network layer with SQLite round-trips. Uses a `tmp_path` fixture for isolated databases.

Additional coverage beyond the network tests:

| Test | What it adds |
|------|-------------|
| `test_add_cp_justification` | Tests CP (conditional proof) justification type, not just SL |
| `test_add_unless_only_justification` | Tests `unless` parameter (outlist-only justification) through the API surface |
| `test_add_justification_no_args_raises` | Validates that calling without `sl`, `cp`, or `unless` raises `ValueError` |
| `test_add_justification_with_namespace` | Namespace prefixing works — result shows `ns:b` not `b` |
| `test_add_justification_persists` | The added justification survives the save/load cycle (round-trip through SQLite) |

## Patterns

**Two-layer testing**: Every feature is tested at both the `Network` (in-memory graph) and `api` (persistence + higher-level interface) layers. The network tests verify propagation logic; the API tests verify serialization, parameter parsing, and the public interface.

**Setup-by-construction**: Each test builds its own small network from scratch rather than sharing fixtures. This keeps tests independent and makes the dependency graph visible in each test body.

**Retract-then-restore pattern**: Several tests follow a `create → retract → add_justification → assert IN` sequence. This exercises the full lifecycle: creation, retraction cascade, and restoration cascade.

**Return-dict inspection**: `add_justification` returns a dict with `node_id`, `old_truth_value`, `new_truth_value`, and `changed`. Tests assert on all of these, treating the return value as a documented API contract.

## Dependencies

**Imports:**
- `reasons_lib.Justification` — the dataclass representing a justification (type, antecedents, outlist)
- `reasons_lib.network.Network` — the in-memory belief graph with propagation
- `reasons_lib.api` — functional API wrapping Network + SQLite storage

**Imported by:** Nothing — this is a leaf test file.

## Flow

A typical test follows this pattern:

1. Create a `Network` (or initialize a SQLite DB via `api.init_db`)
2. Add premise nodes (no justifications — IN by default)
3. Add a derived node with one SL justification depending on a premise
4. Optionally retract a premise to make the derived node OUT
5. Call `add_justification` with a new justification
6. Assert on truth values, justification count, dependent registration, or cascade effects

The propagation engine (BFS inside `Network`) does the heavy lifting — `add_justification` just appends the justification and triggers re-propagation.

## Invariants

1. **A node is IN if ANY justification is valid** — tests like `test_added_justification_keeps_node_in_after_retract` enforce this disjunctive semantics.
2. **SL justification validity requires all antecedents IN and all outlist OUT** — `test_add_justification_with_outlist` exercises the outlist half.
3. **Propagation cascades on restoration** — `test_add_justification_cascades` ensures that downstream dependents are recomputed, not left stale.
4. **`dependents` registration is maintained** — `test_add_justification_registers_dependents` ensures the graph stays wired correctly so future retractions/restorations propagate.
5. **Unknown node IDs raise `KeyError`** — you can't add justifications to nodes that don't exist.
6. **API requires at least one of `sl`, `cp`, or `unless`** — calling with no arguments is a `ValueError`.

## Error Handling

Two explicit error paths are tested:

- `Network.add_justification("ghost", ...)` → `KeyError` with message matching `"not found"`. The network layer validates node existence.
- `api.add_justification("a", db_path=db)` with no `sl`/`cp`/`unless` → `ValueError` with message matching `"Must provide"`. The API layer validates that the caller actually specified a justification.

No tests swallow errors — all error paths are asserted via `pytest.raises`.

---

## Topics to Explore

- [function] `reasons_lib/network.py:add_justification` — The implementation being tested; see how it appends the justification, updates dependents, and triggers propagation
- [function] `reasons_lib/network.py:propagate` — The BFS propagation engine that makes cascading retraction/restoration work
- [file] `reasons_lib/api.py` — The persistence-backed API layer; see how `add_justification` parses `sl`/`cp`/`unless` into a `Justification` and round-trips through SQLite
- [file] `tests/test_outlist.py` — Deeper tests for non-monotonic reasoning (outlist behavior), which `add_justification` must also respect
- [general] `multiple-justification-semantics` — How disjunctive justification (IN if ANY valid) interacts with retraction cascades across the codebase

## Beliefs

- `add-justification-returns-change-dict` — `Network.add_justification` returns a dict with keys `node_id`, `old_truth_value`, `new_truth_value`, and `changed`
- `add-justification-triggers-propagation` — Adding a justification that changes a node's truth value cascades to all transitive dependents via BFS propagation
- `add-justification-registers-dependents` — `add_justification` updates the `dependents` set on both antecedent and outlist nodes so future propagation reaches the target
- `api-add-justification-requires-justification-arg` — `api.add_justification` raises `ValueError` if none of `sl`, `cp`, or `unless` is provided
- `premise-can-receive-justification` — A premise node (initially no justifications) can receive a justification via `add_justification`, giving it a derived backup path

