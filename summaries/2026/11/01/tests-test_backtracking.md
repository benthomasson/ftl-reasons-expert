---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_backtracking.py

# `tests/test_backtracking.py` — Dependency-Directed Backtracking Tests

## Purpose

This file is the test suite for the **dependency-directed backtracking** mechanism in the `ftl-reasons` knowledge base. Its responsibility is to verify that when a contradiction (a "nogood") is detected between two nodes, the system:

1. **Traces backward** through the justification graph to find the *root premises* responsible for the conflicting belief (not just the surface-level nodes in the nogood).
2. **Identifies culprits** — the specific premises whose retraction would resolve the contradiction.
3. **Retracts the least-entrenched premise** rather than an arbitrary node, preserving as much of the knowledge graph as possible.

This is the "smart retraction" path: instead of blindly killing whichever node appears in the nogood, the system reasons about *why* those nodes exist (their justifications) and kills the weakest upstream assumption.

The file encodes a specific production bug it's guarding against: the "physics-speculation bug" (see `TestEntrenchment::test_nogood_retracts_speculation_not_evidence`), where an empirical result (sourced from a simulation) was being incorrectly retracted in favor of a speculative derivation.

## Key Components

### `TestTraceAssumptions`
Tests `Network.trace_assumptions(node_id) -> list[str]`.

**Contract:** Given any node, returns the list of *root premise* IDs that ultimately support it, by walking backward through `Justification` antecedents. A premise traces to itself. A derived node traces to whatever premises feed into it (transitively).

| Test | Graph Shape | Expected Result |
|------|-------------|-----------------|
| `test_premise_traces_to_itself` | single premise `a` | `["a"]` |
| `test_derived_traces_to_premise` | `a → b` | `["a"]` |
| `test_chain_traces_to_root` | `a → b → c` | `["a"]` |
| `test_diamond_traces_to_root` | `a → b, a → c, b+c → d` | `["a"]` |
| `test_multiple_premises` | `a, b → c` | `{"a", "b"}` |
| `test_deep_chain_multiple_roots` | `a → b → d, c → d` | `{"a", "c"}` |
| `test_nonexistent_raises` | — | raises `KeyError` |

### `TestFindCulprits`
Tests `Network.find_culprits(nogood_nodes) -> list[dict]`.

**Contract:** Given a list of node IDs that form a contradiction, returns a list of culprit records, each with:
- `premise`: the ID of the root premise to retract
- `would_resolve`: list of nogood nodes that retraction of this premise would resolve
- Results are **sorted by dependent count ascending** (least disruptive first)

Key assertion in `test_derived_nodes_trace_to_premise`: if the nogood is `["b", "c"]` and `c` is derived from `a`, the culprit for `c`'s side is `a` (the premise), **not** `c` itself.

### `TestBacktrackingInNogood`
Tests the integration of backtracking into `Network.add_nogood(nodes) -> list`.

**Contract:** When a nogood is added between two *active* (IN) nodes:
- A premise (not a derived node) is retracted to OUT.
- Cascading effects propagate: derived nodes that lose their sole support also go OUT.
- A `backtrack` entry is logged in `net.log` with a `culprit` field.
- If either node is already OUT, **no backtracking occurs** and the method returns `[]`.

The test `test_shared_premise_resolves_both` is critical: if a single premise `a` feeds both `b` and `c`, and `nogood(b, c)` is declared, retracting `a` resolves *both* sides simultaneously.

### `TestEntrenchment`
Tests the internal scoring function `Network._entrenchment(node_id) -> float` (or comparable).

**Contract:** The entrenchment score encodes a priority hierarchy for *what to protect*:

1. **Sourced > Unsourced** — a premise with a `source` field (e.g., `"physics-quantum-lattice:results.md"`) is more entrenched than one without.
2. **OBSERVATION > DERIVED** (via `metadata.beliefs_type`) — empirical observations outrank logical derivations.
3. **More dependents = more entrenched** — retracting a node that many other nodes depend on causes more disruption.
4. **Premise (evidence) > Derived (speculation)** — the base test in `test_evidence_premise_more_entrenched_than_derived`.

The production-bug test `test_nogood_retracts_speculation_not_evidence` models the exact scenario: a sourced simulation result must survive; the unsourced speculative chain must be the one retracted.

### `TestBacktrackingAPI`
Tests the **API-layer wrappers** (`reasons.api.trace_assumptions`, `reasons.api.find_culprits`, `reasons.api.add_nogood`) using a temporary SQLite database (`tmp_path` fixture).

**Contract:**
- `api.trace_assumptions("c", db_path=db)` → `{"node_id": "c", "premises": ["a"]}`
- `api.find_culprits(["a","b"], db_path=db)` → `{"culprits": [...]}` with each entry having a `"premise"` key
- `api.add_nogood(["a","b"], db_path=db)` → result includes `"backtracked_to": <node_id>` (not `None`)

These tests verify that the public API correctly delegates to the `Network` internals and serializes the result into JSON-compatible dicts.

## Patterns

- **Graph-shape parameterization via test methods**: Each test constructs a specific DAG (chain, diamond, fork) inline rather than using fixtures, making each test self-contained and readable as a diagram.
- **Set-based assertions for unordered results**: `assert set(result) == {"a", "b"}` — the implementation may return premises in any order, so tests use set comparison where order doesn't matter.
- **Order-based assertions where order matters**: `assert culprits[0]["premise"] == "b"` — when the contract specifies "sorted by dependents ascending," the test checks positional order.
- **Bug-regression naming**: `test_nogood_retracts_speculation_not_evidence` explicitly references the production scenario, serving as documentation of *why* the test exists.
- **API-layer isolation via `tmp_path`**: Each API test gets a fresh SQLite file, preventing cross-test contamination.
- **Log inspection for side effects**: `test_backtracking_logs_culprit` asserts on `net.log` entries, verifying that the operation produces an audit trail, not just a state change.

## Dependencies

### This file imports:
| Import | Role |
|--------|------|
| `pytest` | Test runner; provides `pytest.raises`, `tmp_path` fixture |
| `reasons.Justification` | Dataclass for a single justification edge (`type`, `antecedents`) |
| `reasons.network.Network` | Core graph: nodes, edges, `trace_assumptions`, `find_culprits`, `add_nogood`, `_entrenchment`, `log` |
| `reasons.api` | Public API: `init_db`, `add_node`, `trace_assumptions`, `find_culprits`, `add_nogood` |

### What depends on this file:
Nothing imports a test file. It is a leaf in the dependency graph, run by the pytest harness. However, it **guards** the behavior of `reasons/network.py` (specifically `trace_assumptions`, `find_culprits`, `add_nogood`, `_entrenchment`) and `reasons/api.py` (the API wrappers for those methods).

## Flow

Each test follows a uniform pattern:

```
1. CONSTRUCT  → Create a Network (or init a tmp SQLite DB for API tests)
2. BUILD      → Add nodes with justifications to form a specific DAG
3. ACT        → Call the method under test (trace_assumptions / find_culprits / add_nogood)
4. ASSERT     → Check return value, node truth_values, log entries
```

For `add_nogood` specifically, the internal flow being tested is:

```
add_nogood(["b", "c"])
  │
  ├─ Check: are both nodes IN? (if not → return [])
  │
  ├─ For each node in the nogood:
  │    trace_assumptions(node) → [premise_ids]
  │
  ├─ Collect all candidate premises (the "culprits")
  │
  ├─ For each culprit:
  │    _entrenchment(premise) → score
  │    count dependents
  │
  ├─ Sort culprits by (dependents ASC, entrenchment ASC)
  │
  ├─ Retract the first culprit → set truth_value = "OUT"
  │
  ├─ Cascade: for each node that depended solely on the retracted node → "OUT"
  │
  ├─ Log: {action: "backtrack", value: {culprit: "b", ...}}
  │
  └─ Return list of changed node IDs
```

The `TestBacktrackingAPI` tests add an I/O layer: the same logic runs against a persisted SQLite DB via `api.*` functions, and results come back as JSON-serializable dicts.

## Invariants

1. **Premises are the atomic retraction unit.** `add_nogood` never retracts a derived node directly. It always traces to a premise and retracts that. (Enforced by `test_retracts_premise_not_derived`.)

2. **`trace_assumptions` is transitive and idempotent on premises.** A premise maps to itself; a derived node maps to the set of all root premises in its transitive justification closure. No intermediate derived nodes appear in the result.

3. **Culprit ordering is by disruption cost (ascending).** Fewer dependents → retracted first. This is a soft ordering: the test `test_sorted_by_dependents` asserts positional order.

4. **A node already OUT is inert in a nogood.** If either member of a nogood is already retracted, `add_nogood` is a no-op returning `[]`. No log entry is produced. (Enforced by `test_inactive_nogood_no_backtracking`.)

5. **Entrenchment is a total order.** `_entrenchment` returns a comparable value; the test suite asserts strict inequalities (`>`), not `>=`. Ties are not covered (potential gap).

6. **Source provenance outranks derivation.** A node with a `source` string is always more entrenched than an identical node without one, regardless of other factors. (Enforced by `test_sourced_premise_over_unsourced`.)

7. **The API layer must report which node was backtracked to.** `api.add_nogood` result must have `"backtracked_to"` set to a non-`None` value when backtracking occurred.

## Error Handling

| Scenario | Behavior | Test |
|----------|----------|------|
| `trace_assumptions("missing")` | Raises `KeyError` | `test_nonexistent_raises` |
| `add_nogood` where a node is already OUT | Returns `[]`, no log entry, no exception | `test_inactive_nogood_no_backtracking` |
| `add_nogood` where both nodes are IN | Returns list of changed IDs; logs a `backtrack` entry | `test_backtracking_logs_culprit` |

Notably, there is **no test for `find_culprits` receiving a nonexistent node ID**. The `trace_assumptions` path raises `KeyError`, and since `find_culprits` calls `trace_assumptions` internally, it would propagate the same `KeyError`. This is an untested edge case.

There is also no test for a nogood where *both* nodes are derived from the *same* premise and that premise has zero dependents beyond the two nogood members — the "trivial" case where the premise is the only thing to retract. (Though `test_shared_premise_resolves_both` is close to this.)

---

## Topics to Explore

- [function] `reasons/network.py:trace_assumptions` — The backward-walk algorithm that powers all backtracking; understanding its recursion/stack behavior and how it handles multiple justifications per node is essential for maintaining the invariant that only root premises appear in results.
- [function] `reasons/network.py:_entrenchment` — The scoring heuristic that decides *which* premise gets retracted. The test suite pins down several ordering rules, but the exact formula (weights for source, beliefs_type, dependent count) lives here and is not asserted numerically — only relatively.
- [file] `reasons/repair.py` — The repair/contradiction-detection pipeline that *produces* nogoods fed into `add_nogood`. Understanding how contradictions are detected and formatted clarifies the full lifecycle: detect → nogood → backtrack → retract → cascade.
- [file] `tests/conftest.py` — Shared fixtures and setup for the test suite. The backtracking tests are self-contained (no fixture dependencies beyond `tmp_path`), but other tests in the repo may set up shared state that could interact.
- [general] `nogood-cascade-semantics` — What happens when retracting a premise causes a *second* nogood to activate (a chain reaction). The test `test_cascade_from_retracted_premise` covers direct cascading, but multi-level cascade-with-new-contradiction is not tested here and likely lives in `test_repair.py` or `test_contradictions.py`.

---

## Beliefs

- `add-nogood-retracts-premise-not-derived` — `Network.add_nogood` always retracts a root premise (a node with no justifications), never a derived node, even when the derived node is the one named in the nogood.
- `entrenchment-ranks-source-above-derivation` — A node with a non-empty `source` field always scores higher in `_entrenchment` than an otherwise-identical node without one, making it protected during backtracking.
- `trace-assumptions-returns-roots-only` — The result of `trace_assumptions(node)` contains only premise IDs (nodes without incoming justifications); intermediate derived nodes are never included in the returned list.
- `inactive-nogood-is-noop` — If either member of a nogood already has `truth_value == "OUT"`, `add_nogood` returns an empty list and produces no log entries.
- `api-add-nogood-exposes-backtrack-target` — The public API `api.add_nogood` returns a dict whose `"backtracked_to"` field is non-null whenever a retraction occurred, giving callers programmatic access to which premise was chosen.
