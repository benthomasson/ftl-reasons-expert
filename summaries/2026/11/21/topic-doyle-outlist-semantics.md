---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: The underlying default-logic truth-maintenance model behind inlist/outlist justifications.

# The Default-Logic Truth-Maintenance Model Behind inlist/outlist

## The Underlying Model

This codebase implements a **Justification-based Truth Maintenance System (JTMS)** — explicitly Doyle's (1979) algorithm, as noted in `reasons/network.py:3`. The `inlist`/`outlist` pair is the codebase's reification of Doyle's **Support List (SL) justification**, and it directly encodes the non-monotonic semantics of **default logic**.

### 1. The core truth-value model

Every **node** has a boolean truth value: `IN` (believed) or `OUT` (not believed). This is stored on the node as `truth_value` (`reasons/storage.py`, `nodes` table) and is the only mutable belief state in the system.

Truth values are **not** set directly by users (except for retraction). They are *derived* from the node's justifications. A node with at least one justified support is IN; a node with none is OUT.

### 2. SL justification semantics

A **Justification** is a claim of the form "this node is believed if …". `reasons/__init__.py:7–21` defines the dataclass:

```python
class Justification:
    type: str                    # "SL" or "CP"
    antecedents: list[str]       # inlist: must be IN
    outlist: list[str]           # must be OUT
```

The semantics are stated verbatim in the file's header comments (`reasons/__init__.py:11–15`):

> - SL (Support List): node is IN iff ALL antecedents (inlist) are IN **AND** ALL outlist nodes are OUT
> - The outlist enables non-monotonic reasoning: *"believe X unless Y is IN"*

Formally, for a node `n` with justifications `J`:

```
IN(n)  ⟺  ⋁_j∈J  ( ∀a∈j.antecedents:  IN(a)  )  ∧  ( ∀o∈j.outlist:  OUT(o) )
```

That is: **OR over justifications**, **AND within each justification's inlist**, and **AND over each justification's outlist (negated)**. This is exactly the structure of a Reiter default: `pre : ¬outlist / conclude` — the default fires (conclusion IN) when its prerequisite is IN and its provably-absent condition is OUT.

### 3. Non-monotonicity in practice

The test suite (`tests/test_outlist.py`) demonstrates the semantics directly:

| Test | Scenario | Outcome |
|---|---|---|
| `test_outlist_valid_when_out` | `X` has `SL(outlist=[y])`, `y` is OUT | `X` is IN |
| `test_outlist_invalid_when_in` | `X` has `SL(outlist=[y])`, `y` is IN | `X` is OUT |
| `test_outlist_with_inlist` | `X` has `SL(inlist=[a], outlist=[y])`, `a` IN, `y` OUT | `X` is IN |
| `test_outlist_node_not_in_network` | outlist points to unknown node | `X` is IN (absent = OUT) |

This is **non-monotonic**: adding a belief (`y` asserted) can *retract* another belief (`x`), because `x`'s outlist condition `OUT(y)` is now violated. Monotonic logic cannot do this — new facts only ever add conclusions. Doyle's TMS plus outlists is the mechanism that lets this system "change its mind" when defeater evidence arrives.

The "absent = OUT" rule (`test_outlist_node_not_in_network`) is what makes defaults actually *default*: an unknown defeater is by convention not-believed, so the default fires. Only when the defeater is explicitly asserted does the default retract.

### 4. Propagation (truth maintenance, not just lookup)

The "maintenance" half of TMS is that changes to a node's truth value are **propagated** through the justification graph. `tests/test_outlist.py` exercises this:

- `test_assert_outlist_node_retracts_dependent`: asserting `y` → `x` goes OUT, and the changed set includes `x`
- `test_retract_outlist_node_restores_dependent`: retracting `y` → `x` returns to IN
- `test_outlist_cascade_through_chain`: `y` asserted → `x` OUT → downstream `z` OUT (transitive cascade)
- `test_multiple_outlist_nodes`: `X` unless `{y,z}` requires *both* to be OUT; asserting just `y` suffices to retract `X`

The propagation is queued: `network.assert_node` and `network.retract` return a set of changed node IDs, which callers can observe (see `reasons/pubsub.py`, which publishes "TMS truth-value change events").

### 5. Multiple justifications give "credulous" recovery

`test_alternate_justification_survives_outlist` shows the OR-over-justifications rule: a node with both `SL(outlist=[y])` *and* `SL(inlist=[a])` remains IN when `y` is asserted, because the second justification is still valid. It only goes OUT when `a` is also retracted. This is the standard TMS multiple-justification behavior: alternatives preserve belief, and the node is IN as long as *some* argument survives.

`supporting_justification` (stored on nodes in `reasons/storage.py`) is the single justification the engine chose to justify the current IN state, per `derive.py:_get_depth` which prefers it when present.

### 6. This is what "defeat a justification" means here

`reasons/api.py:1331–1393` implements a `defeat_justification` operation that *appends* a node ID to a justification's `outlist`. Because of the semantics above, doing so is equivalent to saying "this argument no longer holds while this other belief is IN." No reversal is needed — if the defeater is later retracted, the outlist condition becomes satisfied again and the belief is restored (`reasons/api.py:519` describes it as "restoration from outlist — gated beliefs whose blocker is removed"). This is why `add_justification`/`remove_justification` exist as first-class operations (`reasons/network.py:496`, `reasons/network.py:704`) — the graph of justifications *is* the model, not metadata.

### 7. API encoding of defaults

`reasons/api.py:302–334` translates a CLI `--unless y,z` flag into an `SL` justification. Cases:

- `--sl a --unless y` → `SL(antecedents=[a], outlist=[y])` (conditional default)
- `--unless y` alone → `SL(antecedents=[], outlist=[y])` — an **outlist-only justification**, i.e., a "premise that holds unless something is believed" (`reasons/api.py:333–334`). These behave exactly like premises *most of the time*, but remain defeatable, unlike true premises (which have no justifications — see `reasons/review_premises.py:3`).

`tests/test_outlist.py:test_unless_only_no_sl` confirms the outlist-only form produces `type="SL"`.

### 8. Persistence

`reasons/storage.py` stores justifications in the `justifications` table with `antecedents_json` and `outlist_json` columns, and each `node` row carries `truth_value` and `supporting_justification`. `tests/test_outlist.py:TestOutlistPersistence.test_round_trip` proves the semantics survive a SQLite round-trip *and* that propagation still works after reload — the model is state-based, not flow-based.

---

## What's Not Fully Visible in the Observations

The observations give us the *semantics* and *storage* of SL justifications but do not expose the **implementation** of the propagation algorithm itself:

1. The body of `Network.assert_node` / `Network.retract` (and how it recomputes justifications and enqueues updates) is referenced but not shown. The `reasons/network.py:3` comment claims Doyle 1979, but the code that would show dependency-directed propagation is not in the excerpt.
2. The `CP` (conditional proof) justification type appears at `reasons/api.py:331` and in `__init__.py` but is not explained in the visible code.
3. How `supporting_justification` is *selected* among multiple valid justifications (deterministic? lowest index? most-antecedents?) is not visible.

The semantic model is clear; the propagation engine is the open follow-up.

## Topics to Explore

- [file] `reasons/network.py` — The actual TMS propagation algorithm implementing Doyle (1979); contains `assert_node`, `retract`, `add_justification`, `remove_justification`, and the justification-revalidation loop.
- [function] `reasons/network.py:Network.assert_node` — How a truth-value change triggers justification re-evaluation and produces the cascade returned to callers.
- [file] `reasons/__init__.py` — The `Justification` dataclass in full, including how `CP` (conditional proof) justifications differ from `SL` and whether CP supports outlists.
- [function] `reasons/api.py:defeat_justification` — How outlists are extended at runtime to suppress an existing argument without removing it.
- [general] `conditional-proof-justifications` — Explore how `type="CP"` is used (hypothetical reasoning / disjunctive belief) versus `type="SL"` and whether CP justifications carry outlists.

## Beliefs

- `sl-justification-requires-inlist-in-and-outlist-out` — An `SL` justification is satisfied iff all `antecedents` (inlist) nodes are IN and all `outlist` nodes are OUT (`reasons/__init__.py:11–12`).
- `node-in-via-any-satisfied-justification` — A node's truth value is IN if at least one of its justifications is satisfied; it goes OUT only when all justifications fail (`tests/test_outlist.py:TestOutlistWithMultipleJustifications`).
- `outlist-absent-nodes-treated-as-out` — Outlist nodes that do not exist in the network are treated as OUT, so a default justification fires in their absence (`tests/test_outlist.py::test_outlist_node_not_in_network`).
- `outlist-enables-non-monotonic-reasoning` — Because asserting an outlist node can retract a previously-IN dependent, the `outlist` mechanism makes belief revision non-monotonic (`reasons/__init__.py:15`, `tests/test_outlist.py::TestOutlistCascading`).
- `implements-doyle-1979-tms` — `reasons/network.py:3` explicitly states it implements Doyle's (1979) Truth Maintenance System algorithm.
- `outlist-only-justification-is-defeatable-premise` — `add_node(..., unless=...)` without `--sl` creates an `SL` justification with empty antecedents, giving a premise-like default that is still defeatable (`reasons/api.py:333–334`, `tests/test_outlist.py::test_unless_only_no_sl`).
