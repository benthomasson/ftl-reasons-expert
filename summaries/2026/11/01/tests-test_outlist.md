---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_outlist.py

# `tests/test_outlist.py`

## Purpose

This file is the integration test suite for **non-monotonic (default) justifications** in the `ftl-reasons` belief network. It validates the "outlist" mechanism — Doyle's key innovation where an SL justification doesn't only require antecedents to be IN (the *inlist*), but also requires certain nodes to be OUT (the *outlist*). The semantic is: *"Believe X **unless** Y is believed."*

The file owns responsibility for verifying that:
- Outlist evaluation is correct in isolation and in combination with inlist
- Status changes to outlist nodes trigger proper cascading re-evaluation
- Outlist survives persistence and round-trips through storage
- The public API exposes the feature ergonomically (`unless=` keyword)

## Key Components

| Class | Responsibility |
|-------|---------------|
| `TestOutlistBasic` | Verifies the core truth-value logic: OUT list members invalidate the justification; absent nodes are treated as OUT (not believed). |
| `TestOutlistCascading` | Verifies that `assert_node` / `retract` on an outlist member propagates status changes to dependents. |
| `TestOutlistWithMultipleJustifications` | Verifies that a node with *multiple* justifications stays IN as long as *any* one justification holds, even if one is defeated by its outlist. |
| `TestOutlistExplain` | Verifies the `explain()` trace exposes `outlist` and `violated_outlist` fields for debugging. |
| `TestOutlistPersistence` | Verifies the `Storage.save()` / `Storage.load()` SQLite round-trip preserves outlist data and that propagation still works after reload. |
| `TestOutlistAPI` | Verifies the high-level `api.add_node(..., unless="y")` keyword and the combined `sl=` + `unless=` path. |

### Key Contracts

- **`Justification(type="SL", antecedents=[...], outlist=[...])`** — `antecedents` is the inlist (must be IN); `outlist` are node IDs that must be OUT for the justification to fire.
- **`net.assert_node(id)` / `net.retract(id)`** — both return a list of *changed* node IDs (the cascade). Tests assert that outlist-dependent nodes appear in this list.
- **`net.explain(id)`** — returns a list of step dicts; each step includes `"truth_value"`, `"outlist"`, and (when violated) `"violated_outlist"`.
- **`api.add_node(id, desc, unless="y", sl="a", db_path=...)`** — convenience wrapper; `unless` maps to the outlist, `sl` maps to the inlist/antecedents.

## Patterns

1. **Arrange-Act-Assert with explicit state setup.** Each test builds a fresh `Network()`, adds nodes, sets their initial status via `retract()`/`assert_node()`, then asserts the derived node's `truth_value`. No shared state between tests.

2. **Symmetry testing.** For every "assert Y → X goes OUT" test there's a paired "retract Y → X goes IN" test (e.g., `test_assert_outlist_node_retracts_dependent` vs `test_retract_outlist_node_restores_dependent`). This guards against asymmetric cascade bugs.

3. **Boundary / edge cases called out explicitly.** `test_outlist_node_not_in_network` encodes the design decision that *absent = OUT* (i.e., a node not in the network is treated as not-believed, so it satisfies the outlist).

4. **Layered testing.** The same logical feature is exercised at three abstraction levels: raw `Network` → `Storage` persistence → `api` public functions. This catches bugs at whichever layer introduces them.

5. **`tmp_path` fixture** for file-backed tests (persistence, API) — standard pytest pattern avoiding real filesystem pollution.

## Dependencies

**Imports (this file depends on):**
- `reasons.Justification` — dataclass/model for a single justification (type, antecedents, outlist, label)
- `reasons.network.Network` — the belief network engine; owns node status, cascade logic, and `explain()`
- `reasons.storage.Storage` — SQLite persistence layer (`save`/`load`)
- `reasons.api` — public API functions (`init_db`, `add_node`, `retract_node`, `assert_node`, `get_status`)
- `pytest` — test runner + `tmp_path` fixture

**Depended upon by:**
- No other module imports this test file. It is a leaf in the dependency graph.
- CI (`tests/` collection via `pytest`) is the only consumer.

## Flow

```
Test setup
  ├── Network() created in-memory (or Storage(db_path) for persistence tests)
  ├── net.add_node("y", ...)          ← create counter-evidence node
  ├── net.retract("y")                ← set Y to OUT (so outlist is satisfied)
  └── net.add_node("x", justifications=[Justification(outlist=["y"])])
        │
        ▼
Assertion (initial state)
  └── assert net.nodes["x"].truth_value == "IN"
        │
        ▼
Act (trigger cascade)
  └── changed = net.assert_node("y")   ← Y flips to IN
        │
        ▼
Network internal cascade
  ├── Detects Y is now IN
  ├── Finds X's justification whose outlist contains Y
  ├── Marks that justification as violated
  ├── If no other justification holds → X → OUT
  └── Propagates to X's dependents (e.g., Z whose inlist includes X)
        │
        ▼
Assertion (post-cascade)
  ├── assert net.nodes["x"].truth_value == "OUT"
  └── assert "x" in changed
```

For the **persistence test**, the flow adds a `Storage.save(net)` → `Storage.load()` step between setup and assertion, verifying the serialized form retains `outlist` and `label`.

For the **API test**, the flow uses `api.init_db` → `api.add_node(..., unless="y")` → `api.assert_node("y")` → `api.get_status()`, exercising the full DB-backed path.

## Invariants

1. **Absent nodes satisfy outlist.** A node ID in `outlist` that doesn't exist in the network is treated as OUT (`test_outlist_node_not_in_network`). This means outlist is *weaker* than inlist — missing antecedents fail the justification, but missing outlist members pass it.

2. **All outlist members must be OUT.** `test_multiple_outlist_nodes` shows `outlist=["y", "z"]` requires *both* to be OUT. Asserting either one defeats the justification.

3. **Cascade is transitive.** If X depends on (outlist) Y, and Z depends on (inlist) X, then asserting Y cascades through X to Z (`test_outlist_cascade_through_chain`).

4. **Multiple justifications are OR-ed.** A node stays IN if *any* justification is satisfied. Outlist only defeats *that particular* justification (`test_alternate_justification_survives_outlist`).

5. **`explain()` must surface `violated_outlist`.** When a justification is defeated by its outlist, the explain trace includes the specific outlist members that are IN (`test_explain_shows_violated_outlist`).

6. **Persistence is lossless.** After `Storage.load()`, the outlist list, label, and propagation behavior are identical to the in-memory state.

## Error Handling

This is a test file, so "error handling" here means **what failure modes the tests guard against**:

- **No explicit exception handling** — all tests rely on assertion failures (pytest's `assert`) to signal bugs. There are no `try/except` blocks, no `pytest.raises`.
- **The contract being tested is that the network *never* raises** on outlist operations. Status flips (`assert_node`, `retract`) are expected to return a list of changed nodes, not throw. If the network raised instead of cascading, these tests would fail with an unexpected exception.
- **`test_outlist_node_not_in_network`** specifically guards against an `KeyError` or `AssertionError` when the outlist references a non-existent node — the expected behavior is graceful (treat as OUT), not an error.
- **Persistence test** would fail with a data-loss bug (e.g., `j.outlist` is `None` after load) rather than raising, since the assertion `j.outlist == ["y"]` would simply be `False`.

---

## Topics to Explore

- [file] `reasons/network.py` — The core cascade engine; this test file exercises its outlist evaluation and propagation logic, which lives in `Network._evaluate_justification` and `_cascade`.
- [file] `reasons/api.py` — The public API layer; `add_node`'s `unless=` and `sl=` parameters show how the DSL maps to `Justification` construction.
- [file] `reasons/storage.py` — Persistence layer; the outlist serialization format (likely JSON in a SQLite column) is what `TestOutlistPersistence` round-trips.
- [general] `doyle-connection-calculus` — The theoretical foundation (Doyle 1979) behind inlist/outlist semantics; understanding "defeasible" vs "defeater" distinction clarifies why absent = OUT.
- [file] `tests/test_hide_out.py` — Likely tests the inverse behavior (hiding/retracting nodes) and may interact with outlist logic when outlist nodes are retracted.

---

## Beliefs

- `outlist-absent-node-satisfies-condition` — A node ID in `Justification.outlist` that does not exist in the network is treated as OUT, so it does not defeat the justification (unlike a missing antecedent in `antecedents`).
- `outlist-uses-conjunctive-semantics` — All members of `Justification.outlist` must be OUT for the justification to hold; a single IN member defeats the entire justification.
- `multiple-justifications-are-disjunctive` — A node with multiple `Justification` entries remains IN as long as at least one justification is satisfied; outlist only defeats the specific justification it belongs to.
- `explain-exposes-violated-outlist` — `Network.explain()` includes a `violated_outlist` key (list of IN node IDs) in the step dict when a justification is defeated by its outlist, but only `outlist` (the declared list) when it is satisfied.
- `api-unless-maps-to-outlist` — The `api.add_node(..., unless="y")` keyword parameter is syntactic sugar that creates a `Justification(type="SL", outlist=["y"])`; combining with `sl="a"` produces `antecedents=["a"], outlist=["y"]`.
