---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Important for understanding justifications, antecedents, outlists, and why depth is structural.

# Justifications, Antecedents, Outlists, and Why Depth Is Structural

## 1. The core data model — `Justification`

The single source of truth lives in `reasons/__init__.py:7`. A `Justification` has three fields that matter here (docs at lines 11–15, fields at lines 20–23):

| Field | Role | Semantics |
|---|---|---|
| `type` | "SL" or "CP" | Tag that distinguishes support-list vs. conditional-proof justifications |
| `antecedents: list[str]` | **inlist** | "must be IN" (`reasons/__init__.py:20`) |
| `outlist: list[str]` | **outlist** | "must be OUT" (`reasons/__init__.py:21`) |

A node can carry **multiple** justifications. The validity rule, spelled out at `reasons/__init__.py:11`, is:

> "node is IN iff ALL antecedents (inlist) are IN and ALL outlist nodes are OUT"

Read carefully: that's **OR at the node level** (one valid justification is enough) but **AND at the justification level** (every antecedent IN, every outlist OUT). The tests make this explicit:

- `test_alternate_justification_survives_outlist` (`tests/test_outlist.py`) — one justification is defeated by `y` going IN, but a second valid justification keeps the node IN.
- `test_multiple_outlist_nodes` — `outlist=["y", "z"]` means *both* `y` and `z` must be OUT; asserting either one defeats the justification.

The validity combination is implemented at `reasons/api.py:1590–1592`, where `outlist_ok = all(...)` is computed against `j.outlist` before the antecedents are combined.

## 2. Antecedents (inlist) — monotonic support

Antecedents are the *positive premises* a claim depends on. They are:

- Parsed from the API's `sl=` / `cp=` arguments (`reasons/api.py:323`, `reasons/api.py:330`, `reasons/api.py:440`, `reasons/api.py:443`).
- Namespace-resolved when the node lives in a namespace (`reasons/api.py:367`, `reasons/api.py:454`).
- Additionally pre-seeded with `ns:active` for namespaced nodes, so that a justification only holds while the namespace is "active" (`reasons/api.py:350–360`).
- Stored as `antecedents_json` in the `justifications` table (`reasons/storage.py`, `SCHEMA`).

Two important details:

**`any_mode` reverses AND into OR.** At `reasons/api.py:324–326`, if `any_mode=True` and `sl="a,b,c"`, the code expands it into three separate SL justifications, each with one antecedent. Since the node is IN if *any* justification holds, the net effect is OR-over-antecedents — the caller asked for "A or B or C implies X" but delivered it using AND-at-justification-level wiring. `max_premises` (`reasons/api.py:395`) is used to size the resulting expansion for downstream display.

**SL vs. CP only differ by tag at this layer.** At `reasons/api.py:328` vs `reasons/api.py:331`, both branches call `Justification(...)` with the same lattice of once-antecedents/outlist; only `type="SL"` vs `type="CP"` distinguishes them. `tests/test_add_justification.py::test_add_cp_justification` confirms the CP tag round-trips through SQLite and shares validity semantics — so CP is a label, not a separate inference rule, at least in the code paths visible here.

## 3. Outlists — non-monotonic defeat

The outlist is what makes the system *defeasible*. From `reasons/__init__.py:15`:

> "The outlist enables non-monotonic reasoning: 'believe X unless Y is believed.'"

Concrete behaviors verified in `tests/test_outlist.py`:

- `test_outlist_valid_when_out` — `y` is retracted (OUT), so X with `outlist=["y"]` is IN.
- `test_outlist_invalid_when_in` — `y` is IN, so X is OUT.
- `test_outlist_node_not_in_network` — an outlist member that doesn't exist is treated as OUT. Absence = not believed.
- `test_assert_outlist_node_retracts_dependent` / `test_retract_outlist_node_restores_dependent` — asserting or retracting `y` cascades to `x` in either direction.

The crucial wiring is that **outlist nodes get the target registered as a dependent**, not just an antecedent. `tests/test_add_justification.py::test_add_justification_registers_dependents` proves it:

```python
net.add_node("e", "Enemy")
net.add_justification("b", Justification(type="SL", antecedents=["a"], outlist=["e"]))
assert "b" in net.nodes["e"].dependents
```

This is why propagation can re-evaluate `b` when `e` changes — the runtime treats outlist membership with the same change-tracking relationship as antecedents. Without that registration, outlists would only be checked when `b` itself was touched, and defeasibility would break.

The API surface for this is the `unless=` parameter, parsed at `reasons/api.py:320` and `reasons/api.py:437`. Notice at `reasons/api.py:332–334` that **`unless` with no `sl`/`cp` produces an SL justification with `antecedents=[]`** — an empty-inlist, non-empty-outlist justification. That's a *default*: the node holds unconditionally *unless* a blocker is asserted. `tests/test_outlist.py::test_unless_only_no_sl` asserts exactly this ("premise-like node that holds unless something is believed").

The runtime also detects the reverse direction: `reasons/api.py:519` restores outlist-gated beliefs when a blocker is removed, and `reasons/api.py:586` reports outlist-gated beliefs that lose justification when a blocker goes IN. Both are surfaced in cascade output.

## 4. What "depth" means — and why it's structural

There are **two distinct depths** in the codebase and conflating them leads to confusion:

### 4a. `_node_depth` — structural depth (`reasons/api.py:3368–3387`)

```python
def _node_depth(nid, net, memo=None):
    """Compute depth of a node: 0 for premises, max(antecedent depths)+1 for derived."""
    ...
    max_d = max(max_d, _node_depth(a, net, memo))
```

This is what "depth is structural" refers to. Its definition has three properties that make it *structural* rather than *operational*:

1. **Base case is topological.** Premises (nodes with no justifications) are depth 0. Full stop. No truth value, no log history, no entrenchment score is consulted.
2. **Recurrence only touches antecedents.** `depth(X) = max(depth(a) for a in antecedents) + 1` (`reasons/api.py:3387`). Outlists are ignored. Whether `X` is currently IN or OUT is ignored. The number depends exclusively on the shape of the justification DAG.
3. **It is memoized** (`memo` parameter at `reasons/api.py:3368`, passed through at `reasons/api.py:3387`, instantiated at `reasons/api.py:3446`). Memoization is only sound because the function is a pure function of the graph — the same value for a given node regardless of when you ask.

This is the number used by the query filter API: `min_depth` and `max_depth` at `reasons/api.py:3423–3426`, applied via `d = _node_depth(nid, net, memo)` at `reasons/api.py:3468`. A caller can ask "which claims are at conservative derivation depth 2 or deeper?" without caring whether any of those claims are currently IN. That's only possible because depth is structural.

It also gives a **canonical ordering** for explanation output: shallower nodes are closer to axioms, deeper nodes are more derived. Retracting a premise does not change any node's depth — the label is stable across all state changes — which is exactly what you want for a presentation layer that shouldn't shuffle when the world updates.

### 4b. `_cascade_depth` — operational depth (`reasons/api.py:737–751`)

```python
def _cascade_depth(net, target_id: str, retracted_id: str) -> int:
    ...
    current_id, depth = queue.popleft()
    ...
    return depth + 1
    queue.append((dep_id, depth + 1))
```

This is a BFS over the **dependents** graph from the node that just changed (`retracted_id`) to each affected node, returning the hop count. It is used literally at every cascade-report call site to sort results nearest-first:

```python
# reasons/api.py:568–569 (and again at 635–636 and 724–725)
retracted.sort(key=lambda c: (c["depth"], c["id"]))
restored.sort(key=lambda c: (c["depth"], c["id"]))
```

This is not structural because it depends on *which* node triggered the operation. Retracting `a` versus retracting `b` produces different `_cascade_depth` values for the same downstream node. It's the radius of the ripple, not the altitude of the proof.

### 4c. Why the distinction matters

The codebase can afford to compute `_node_depth` once and cache, and refilters under `min_depth`/`max_depth` cheaply, precisely because it does *not* encode runtime state. If depth were computed from the "current number of retracted nodes under this one" or from propagation order, the following would all break:

- Filtering by `min_depth`/`max_depth` in the query API (`reasons/api.py:3423–3426`) would be a runtime snapshot, not a query dimension.
- Memoization (`reasons/api.py:3446`) would be unsound — the answer changes on every propagation, invalidating the cache.
- Explanation of *why* a claim is at level 2 (two derivations from axioms) would become an explanation of *how it happened to be touched recently*.

The cascade engine specifically wants `_cascade_depth` because users read cascade output in ripple order ("the thing you changed, then its direct dependents, then their dependents"). You want the operational number for that. But anything that describes the *shape* of the knowledge — level filters, explanation ordering by entailment — needs the structural number.

`_node_depth` is therefore the honest answer to "why is depth structural": because it is defined purely on the justification DAG, with premises anchored at 0 and each derivation step adding exactly 1, with no reference to any mutable state.

## 5. What's missing from the observations

A few things were inferred from tests and call sites rather than read directly, and are worth flagging:

- The exact body of the validity check at `reasons/api.py:1590–1592` — only `outlist_ok` construction is shown, not the antecedent combination.
- The propagation engine in `reasons/network.py` (`assert_node`, `retract`, the dependent walk) is described only via its tests. That's the code that turns the static `Justification` model into a changing truth value.
- Whether `CP` justifications ever behave differently from `SL` at propagation time. From `reasons/api.py:328` vs `331` and the test asserting `type == "CP"`, they appear to differ only by tag — but the propagation path could branch on `j.type` somewhere not in the observations.
- The `content_hash` field on `Justification` (`reasons/storage.py` schema) — computed elsewhere and presumably used for deduplication or change detection.
- The `_node_depth` memoization strategy across multiple top-level queries — the instantiation at `reasons/api.py:3446` is per-query, so depth is recomputed across separate calls; that's fine but worth knowing.

## Topics to Explore

- [file] `reasons/network.py` — The propagation engine (`assert_node`, `retract`, `add_nogood`) that actually reads antecedents and outlists to derive truth values; the observations only show its tests.
- [function] `reasons/api.py:_node_depth` — The structural-depth implementation and its memoization contract; the anchor for understanding why depth is a graph invariant.
- [function] `reasons/api.py:_cascade_depth` — The operational BFS depth used for ordering cascade reports; contrast with `_node_depth`.
- [function] `reasons/api.py:add_node` — The `sl`/`cp`/`unless`/`any_mode` constructor that turns API strings into one or more `Justification` objects; central to the antecedent/outlist surface.
- [general] `cp-vs-sl-semantics` — Whether `type="CP"` justifications behave differently from `type="SL"` at propagation, backtracking, or explanation time (observations suggest only a tag difference).

## Beliefs

- `justification-valid-iff-antecedents-in-and-outlist-out` — A `Justification` is valid only when every antecedent is IN **and** every outlist node is OUT; a missing outlist node counts as OUT.
- `node-in-if-any-justification-valid` — A derived node is IN when at least one of its justifications is valid, so multiple justifications act as alternates (OR at node level).
- `outlist-nodes-register-as-dependents` — Nodes listed in an outlist have the target registered in their `dependents` set, which is why asserting an outlist node cascades to its target.
- `node-depth-is-structural` — `_node_depth` anchors premises at 0 and computes derived depth as `max(antecedent depths) + 1`, consulting neither truth value nor outlist.
- `cascade-depth-is-operational` — `_cascade_depth` is a BFS hop count from the just-changed node to each affected dependent, and is the sort key for every cascade report (`reasons/api.py:568–569`, `635–636`, `724–725`).
