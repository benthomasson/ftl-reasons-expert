---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_network.py

# `tests/test_network.py`

## 1. Purpose

This file is the **unit test suite for the dependency network and truth-maintenance propagation algorithm** — the core engine of the `ftl-reasons` project. It validates that the `Network` class correctly implements a **Doyle-style Truth Maintenance System (TMS)** where nodes are either `IN` (believed) or `OUT` (not believed), and state changes propagate through justifications and nogoods.

The responsibility it owns is ensuring that:
- Node lifecycle (add, retract, restore) works correctly
- Propagation cascades through chains, diamonds, and multi-antecedent dependencies
- "Sticky" retraction (pinning) prevents unwanted resurrection
- Contradiction detection (nogoods) retracts exactly one conflicting node
- The explain trail is complete and accurate
- Edge cases (dangling dependents, duplicate adds, non-existent nodes) are handled

## 2. Key Components

| Class / Group | What It Tests |
|---|---|
| `TestAddNode` | `net.add_node()` — creation of premises and derived nodes, duplicate rejection, dependent registration |
| `TestRetraction` | `net.retract()` — cascade propagation, no-op on already-OUT, **pinning** (`_retracted` metadata), and the critical "alternate justification keeps node IN" case |
| `TestRestoration` | `net.assert_node()` — re-asserting a premise restores its entire downstream chain |
| `TestMultipleAntecedents` | SL justifications with multiple antecedents (logical AND) |
| `TestNogood` | `net.add_nogood()` — contradiction detection, idempotency, and nogood recording |
| `TestExplain` | `net.explain()` — human-readable trace of why a node is IN or OUT |
| `TestBeliefSet` | `net.get_belief_set()` — returns the set of all IN node IDs |
| `TestLog` | `net.log` — audit trail of add/retract/propagate/warn actions |
| `TestDiamondDependency` | Diamond graph (A→B, A→C, B+C→D) with sticky retraction semantics |
| `TestDanglingDependents` | Graceful handling of `dependents` referencing nodes that don't exist in `net.nodes` |

**Key types from imports:**
- `Node` — has `.truth_value` (`"IN"`/`"OUT"`), `.justifications`, `.dependents` (a set of node IDs), `.metadata` (dict)
- `Justification` — `type` (e.g. `"SL"`), `antecedents` (list of node IDs), optional `label`
- `Nogood` — `.nodes` (list of conflicting IDs), `.id` (deterministic: `"nogood-a-d"`)
- `Network` — the SUT with `.nodes` (dict), `.nogoods` (list), `.log` (list of dicts)

## 3. Patterns

- **Grouped test classes**: Each `class Test*` maps to a behavioral concern, not a code path. This mirrors the feature-level decomposition of the `Network` API.
- **Arrange-Act-Assert with explicit graph setup**: Every test builds a minimal DAG inline (2–4 nodes), performs one mutation, then asserts on the full state. No shared fixtures beyond `Network()` — each test is fully independent.
- **Sticky-retraction contract tests**: `test_retract_already_out_pins_retracted` and `test_diamond_with_retracted_intermediate` encode a critical invariant: *explicit retraction is sticky and survives `recompute_all()`*. These are the trickiest semantics in the system and get dedicated multi-step tests.
- **Negative-path testing**: Duplicate adds, non-existent node references, and already-settled states all assert specific exception types (`ValueError`, `KeyError`) with message matching.
- **Audit-trail assertions**: `TestLog` and `test_propagate_logs_dangling_warning` verify that the side-channel `net.log` is populated correctly, ensuring observability for production debugging.

## 4. Dependencies

**Imports (this file → project):**
- `pytest` — test runner, `pytest.raises` for exception assertions
- `reasons` (package `__init__`) — re-exports `Node`, `Justification`, `Nogood` data types
- `reasons.network` — the `Network` class under test

**Imported by (project → this file):**
- Nothing imports a test file. It is a leaf in the dependency graph, executed by the test runner (`pytest` / CI workflow in `.github/workflows`).

**Implicit dependency on `tests/conftest.py`**: May provide shared fixtures (e.g., DB cleanup, temp storage) that these tests currently don't need but could pick up.

## 5. Flow (Control Flow)

Each test follows the same micro-lifecycle:

```
1. net = Network()                          # fresh state, no shared state between tests
2. net.add_node(...) × N                    # build the DAG (premises first, then derived)
3. [optional] net.retract(...)              # mutate state
4. assert on net.nodes[...].truth_value     # verify IN/OUT
5. [optional] net.assert_node(...)          # restore
6. assert on full graph state               # verify cascade
```

The propagation flow being tested (inside `Network`) is:
- `retract(id)` → set node OUT → walk `node.dependents` → for each dependent, check if *all* justifications still have all-antecedents-IN → if not, retract that dependent → recurse
- `assert_node(id)` → set node IN → walk dependents → if a dependent was OUT and now has a fully-satisfied justification, restore it → recurse
- The **pin check**: if `node.metadata["_retracted"]` is `True`, skip restoration even if antecedents are back

## 6. Invariants Enforced by These Tests

1. **At most one truth value**: Every node is either `"IN"` or `"OUT"`, never both.
2. **Sticky retraction**: Once explicitly retracted, a node stays OUT until explicitly asserted, regardless of antecedent changes or `recompute_all()`.
3. **Alternate-justification survival**: A node with ≥2 justifications stays IN as long as *one* justification is fully satisfied.
4. **Nogood exclusivity**: When a nogood becomes active (all members IN), exactly one member is retracted.
5. **Idempotent no-ops**: Retracting an already-OUT node (without pin) or asserting an already-IN node returns an empty `changed` list.
6. **Dangling tolerance**: Propagation must not crash on dependents that don't exist in `net.nodes`; it logs a warning instead.
7. **Deterministic nogood IDs**: `nogood-{a}-{b}` format from sorted node IDs.

## 7. Error Handling

| Scenario | Expected Behavior | Test |
|---|---|---|
| `add_node` with existing ID | Raises `ValueError("already exists")` | `test_add_duplicate_raises` |
| `retract` / `assert_node` / `explain` / `add_nogood` with unknown ID | Raises `KeyError("not found")` | 4 separate tests |
| Dangling dependent in propagation | No crash; appends `{"action": "warn", "target": ..., "value": "...dangling..."}` to `net.log` | `test_propagate_logs_dangling_warning` |
| Nogood where one node is already OUT | No-op; returns empty list; other node stays IN | `test_nogood_inactive_when_one_already_out` |

The tests never assert on exception *messages* beyond a `match=` regex fragment, keeping the contract at the type level.

---

## Topics to Explore

- [file] `reasons/network.py` — The implementation under test; understanding `_propagate`, `_check_justifications`, and the `_retracted` pin logic is essential before modifying these tests
- [file] `tests/test_network_cache.py` — Sibling test file that likely covers caching/optimization of the same propagation algorithm; reveals whether caching invariants are enforced separately
- [file] `reasons/contradictions.py` — Works alongside `Network` for nogood generation; understanding how nogoods are *produced* (vs. consumed here) completes the picture
- [general] `doyle-tms-semantics` — The Doyle 1979 TMS paper defines the formal model (IN/OUT, nogoods, "no-joint" justifications); the `_retracted` pin is an extension beyond classical TMS worth understanding in context
- [file] `tests/test_dangling_dependents.py` — A dedicated test file for dangling-reference scenarios; compare coverage with the last test class here to see if edge cases are split across files

---

## Beliefs

- `retract-pins-node-permanently` — Once `net.retract(id)` is called on a node, `node.metadata["_retracted"]` is set to `True`, and neither `assert_node` on antecedents nor `recompute_all()` will restore the node to IN until an explicit `assert_node(id)` is called
- `network-propagation-respects-alternate-justifications` — A derived node with multiple `Justification` entries remains IN if at least one justification has all antecedents IN, even after retracting antecedents of other justifications
- `nogood-id-is-deterministic` — `Nogood.id` is always the concatenation `"nogood-" + "-".join(sorted(nodes))`, making it stable for deduplication and external reference
- `network-log-is-append-only-audit-trail` — `net.log` is a list that only grows (add/retract/propagate/warn entries); no test asserts log entries are ever removed or reordered, and production code must not truncate it
- `dangling-dependents-are-warned-not-raised` — Propagation encounters a dependent ID absent from `net.nodes` by logging a `"warn"` entry rather than raising, ensuring a single corrupted reference cannot halt the entire cascade
