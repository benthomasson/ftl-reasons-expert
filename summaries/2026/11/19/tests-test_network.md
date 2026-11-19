---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_network.py

# `tests/test_network.py` — Dependency Network & Propagation Tests

## Purpose

This file is the behavioral contract for `reasons.network.Network`, the truth-maintenance core of the `reasons` package. It tests Doyle-style TMS (Truth Maintenance System) semantics: nodes with justifications, truth values (`IN`/`OUT`), retraction/restoration cascades, nogoods (contradiction sets), explanation traces, and audit logging.

It is the **specification** for how dependents react when premises change — the deepest invariant of the belief-reasoning system.

## Key Components

### Imports
- `pytest` — for `pytest.raises`, `pytest.fixture`-less style.
- `Node`, `Justification`, `Nogood` from `reasons` (the public data model).
- `Network` from `reasons.network` (the graph/engine under test).

Note: `Node` and `Nogood` are imported but only `Justification` is directly instantiated in test bodies (via `Justification(type="SL", antecedents=[...], label=...)`). `Node` and `Nogood` are effectively imported for type-context / re-export clarity.

### Test classes and their contracts

| Class | Focus | Key contract enforced |
|---|---|---|
| `TestAddNode` | Node creation API | `add_node(id, desc, justifications=[...])` returns `Node`; duplicate IDs raise `ValueError("already exists")`; antecedents register the new node in `dependents`. |
| `TestRetraction` | `retract(id)` | Returns list of changed node IDs; cascades to dependents lacking alternate justifications; already-OUT retract is a no-op returning `[]` but **sets `metadata["_retracted"] = True`** as a sticky pin. Missing node → `KeyError("not found")`. |
| `TestRestoration` | `assert_node(id)` | Inverse of retract; returns changed nodes; already-IN is no-op (`[]`); missing → `KeyError`. |
| `TestMultipleAntecedents` | Conjunctive SL | `SL(a,b,c)` requires **all** antecedents IN. Any single retract → OUT; re-assert → IN. |
| `TestNogood` | `add_nogood([ids])` | Idempotently retracts ≥1 member (does not raise even if one is already OUT); `Nogood.id` = `"nogood-<a>-<b>"`; `Nogood.nodes` preserves input order; missing node → `KeyError`. |
| `TestExplain` | `explain(id)` | Returns `list[dict]` with keys `node`, `truth_value`, `reason` (`"premise"`, `"retracted premise"`, etc.), `label`, `failed_antecedents`. Traces recursively through justifications. |
| `TestBeliefSet` | `get_belief_set()` | Returns iterable of IN node IDs only. |
| `TestLog` | `net.log` | Append-only ledger of `{action, target, value?}` events. Actions include `"add"`, `"retract"`, `"propagate"`, `"warn"`. |
| `TestDiamondDependency` | Diamond graph & `recompute_all()` | Propagation across diamond topology; `_retracted` pin survives `recompute_all()`; explicitly re-asserting clears the pin. |
| `TestDanglingDependents` | Robustness | `dependents` containing unknown IDs (simulated corruption) must not crash `_propagate` — instead emits `{"action": "warn", "value": "...dangling..."}`. |

## Patterns

- **Arrange/Act/Assert per method**, organized by behavior facet — standard pytest class-scoped test grouping.
- **Fixture-free** — every test builds its own `Network()` from scratch. This validates that construction is cheap and side-effect-free (a precondition for the package's use in derived reasoning).
- **Comment-as-spec** — docstrings state the graph shape (e.g. `"A → B → C: retract A → both B and C go OUT"`) before the code, so failures carry their intent.
- **Sticky-state testing** — `test_retract_already_out_pins_retracted` and `test_diamond_with_retracted_intermediate` explicitly guard the "sticky retract" semantics that distinguish this engine from a pure recompute-from-justifications TMS. This is a deliberate design choice — retracted nodes must not resurrect when precedent conditions return.
- **Log-as-observable** — tests don't inspect private fields (e.g. `_propagate`) directly; they read `net.log`, ensuring the audit trail is a first-class API.
- **Error-surface asymmetry**: mutation-era errors (`add_node`, `retract`, `assert_node`, `add_nogood`, `explain`) raise typed exceptions; internal graph inconsistency (`dependents` pointing to missing nodes) is *warned* via the log and swallowed.

## Dependencies

**Imports:**
- `reasons` package public API — `Node`, `Justification`, `Nogood` must exist at `reasons/__init__.py`.
- `reasons.network.Network` — the class under test.

**Implicitly depends on** (must exist and behave as contracted):
- `Network.add_node`, `Network.retract`, `Network.assert_node`, `Network.add_nogood`, `Network.explain`, `Network.get_belief_set`, `Network.recompute_all`, `Network._propagate` (indirectly, via `retract`).
- `Node` properties: `.truth_value`, `.justifications`, `.dependents`, `.metadata`.
- `Justification` constructor with `type`, `antecedents`, optional `label`.
- `Nogood` properties: `.nodes`, `.id`.

**Imported by** — nothing (test files are leaves). However, `tests/test_network_cache.py` is a sibling that presumably exercises caching in the same module.

## Flow

Taken as a whole, tests follow this data flow:

1. **Construct** an empty `Network()`.
2. **Seed premises** via `add_node(id, desc)` — each starts `IN` with no justifications.
3. **Add derived nodes** with `Justification(type="SL", antecedents=[...])` — truth value is *computed* at insertion time (IN if antecedents IN, else OUT).
4. **Mutate** via `retract(id)` / `assert_node(id)` / `add_nogood([...])` — each returns a list of changed node IDs (the *delta*).
5. **Observe** via `net.nodes[id].truth_value`, `net.get_belief_set()`, `net.explain(id)`, `net.log`.
6. **Reset** via `net.recompute_all()` — replays justification logic from scratch, honoring `_retracted` pins.

Internally, `retract(a)` in the engine:
- Sets `a.truth_value = "OUT"` (and `_retracted = True` if not already OUT).
- Calls `_propagate(a)` which walks `a.dependents`, logs `{"action": "propagate", "target": <dep>}` for each, and re-evaluates each dependent's justifications.
- Recurses through the dependents graph (validated by `test_retract_cascades_through_chain`).
- Returns the accumulated delta of changed IDs.

## Invariants

- **Unique IDs**: `add_node` on an existing ID must raise `ValueError`.
- **Referential validity on mutation**: any ID passed to `retract`, `assert_node`, `add_nogood`, or `explain` must exist, else `KeyError("not found")`.
- **SL is conjunctive**: every antecedent in a `Justification(type="SL", antecedents=[...])` must be `IN` for the node to be `IN`.
- **Alternate justifications are OR'd**: a node is `IN` if *any* justification is satisfied. Retraction only cascades when none remain satisfied (`test_retract_does_not_cascade_with_alternate_justification`).
- **Sticky retraction (`_retracted` pin)**: an explicit `retract()` of an already-OUT node sets `metadata["_retracted"] = True`; that pin must survive `assert_node` on antecedents AND `recompute_all()`. Only an explicit `assert_node(id)` on the pinned node clears it (`test_diamond_with_retracted_intermediate`).
- **Nogood idempotence**: `add_nogood` on a set already containing an OUT node makes no changes (`changed == []`) — it defeats the purpose of detecting a contradiction only when all members are IN.
- **Dependents registration**: `add_node` with a justification MUST append the new node's ID to each antecedent's `dependents` set (`test_dependents_registered`).
- **Log append-only**: every state change is recorded with an `action` and `target`; warnings use `action == "warn"`.

## Error Handling

- **Raise (loud failures)** for programmer errors:
  - `ValueError("already exists")` on duplicate `add_node`.
  - `KeyError("not found")` on missing-ID operations.
- **Log-and-continue (soft failures)** for graph-integrity bugs:
  - Dangling `dependents` entries encountered during `_propagate` emit `{"action": "warn", "target": "<id>", "value": "...dangling..."}` and are skipped — `test_propagate_skips_dangling_dependent` asserts no exception; `test_propagate_logs_dangling_warning` asserts exactly one warning.
- **No swallowed exceptions on user input**: all the "bad input" paths are tested to raise, not to silently succeed. The only tolerated anomaly is internal state corruption, and even that is surfaced via the log.

---

## Topics to Explore

- [file] `reasons/network.py` — The actual `Network` implementation; verify how `_propagate`, `recompute_all`, and `_retracted` pinning are implemented and where `_propagate` recurses.
- [function] `reasons/network.py:Network.add_nogood` — Check the selection heuristic: which member of the nogood gets retracted (deterministic? lexicographic? first IN?) — the tests only assert that "one" goes OUT, not which.
- [file] `tests/test_network_cache.py` — Sibling test file; presumably verifies memoization of `recompute_all`/`_propagate`, which has correctness implications for the sticky-state tests here.
- [function] `reasons/network.py:Network.explain` — The trace format (`steps[*]["reason"]`, `failed_antecedents`, `label`) is contractually fixed here; any consumers (CLI, explain endpoint) rely on those exact keys.
- [general] `doyle-tms-semantics` — Doyle's original TMS distinguishes "in/out" from "retracted/pinned"; understanding which tests encode TMS proper vs. this project's sticky extensions will clarify `_retracted` design.

## Beliefs

- `network-requires-justification-type-sl` — `Network.add_node` accepts justifications of type `"SL"`; the test suite only exercises this type, so other types are untested contracts.
- `retract-returns-changed-node-ids` — Both `retract` and `assert_node` return a `list` of changed node IDs and return `[]` when no state changes.
- `retract-sets-retracted-metadata-pin` — Calling `retract` on an already-OUT node returns `[]` but sets `node.metadata["_retracted"] = True`, which prevents resurrection under `assert_node` on antecedents and `recompute_all`.
- `nogood-id-follows-nogood-a-b-pattern` — `Nogood.id` is constructed as `"nogood-" + "-".join(node_ids)` preserving input order.
- `dangling-dependents-produce-warn-log-entries` — A `dependents` reference to an unknown node produces exactly one `{"action": "warn"}` log entry with `value` containing `"dangling"` and does not raise.
