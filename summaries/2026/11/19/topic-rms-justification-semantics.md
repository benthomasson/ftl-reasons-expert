---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: How `SL` justifications with `outlist` implement `unless` / defeasibility in this RMS; this determines what "computed as OUT" actually means for the cycle fallback.

## How `SL` + `outlist` implements `unless`

An `SL` justification is not just “antecedents imply conclusion.” It carries two lists:

- `antecedents` / inlist: all must be `IN`
- `outlist`: all must be `OUT`

That contract is documented directly in `reasons/__init__.py:11-21`:

```text
- SL (Support List): node is IN iff ALL antecedents (inlist) are IN
  AND ALL outlist nodes are OUT
...
The outlist enables non-monotonic reasoning: "believe X unless Y is believed."
```

So for a justification:

```python
Justification(type="SL", antecedents=["a"], outlist=["y"])
```

the support condition is effectively:

```text
holds(j) = all(truth(a) == "IN" for a in j.antecedents)
           and
           all(truth(o) == "OUT" for o in j.outlist)
```

A node is then `IN` if at least one of its justifications holds; it is `OUT` only when none of its justifications currently hold. This is why `tests/test_outlist.py::TestOutlistWithMultipleJustifications::test_alternate_justification_survives_outlist` shows that one failed `unless` justification does not retract the node if another justification still holds.

The actual outlist check is visible around `reasons/api.py:1590-1592`:

```python
outlist_ok = all(
    ...
    for o in j.outlist
)
```

That is the implementation point where “all outlist nodes must be OUT” is enforced.

## How `unless` is created through the API

The API maps `unless` directly into `Justification.outlist`.

In `reasons/api.py:300-334` and again around `reasons/api.py:423-466`, `unless` is documented as:

```text
Comma-separated outlist IDs (must be OUT for justification to hold)
```

and parsed into:

```python
outlist = [o.strip() for o in unless.split(",") if o.strip()] if unless else []
```

Then `add_node` / `add_justification` creates SL justifications with that outlist, e.g.:

```python
justifications.append(
    Justification(type="SL", antecedents=antecedents, outlist=outlist, label=label)
)
```

Important special case: `unless` without `sl` creates a premise-like default:

```python
# Outlist-only justification (no inlist) — premise that holds unless something is believed
justifications.append(
    Justification(type="SL", antecedents=[], outlist=outlist, label=label)
)
```

See `reasons/api.py:332-334`. That is the code path behind the test `test_unless_only_no_sl` in `tests/test_outlist.py`.

## How defeat / defeasibility reuses `outlist`

Defeat is not a separate truth-value mechanism. `defeat_justification` implements defeat by creating a normal defeater belief and adding its ID to the target justification’s `outlist`.

Relevant code:

- `reasons/api.py:1322` defines `defeat_justification`
- `reasons/api.py:1331-1334` explains that it defeats a justification by adding a defeater belief to its outlist
- `reasons/api.py:1392-1393` appends the defeater ID to `justification.outlist`
- the same pattern appears again around `reasons/api.py:1526-1527`

So if `belief` is supported by `SL(p)` and you defeat it, the system creates an `IN` defeater node and mutates the justification to something like:

```python
Justification(type="SL", antecedents=["p"], outlist=["over-generalizes-belief-j0"])
```

Because the defeater is `IN`, the outlist condition fails, so the justification no longer holds. If that was the only justification, the node goes `OUT`.

Retracting the defeater makes the outlist node `OUT`, so the original justification can hold again. That restoration behavior is tested in `tests/test_defeat_justification.py::test_defeat_restoration_on_retract`.

This matches the comments in `reasons/api.py:519` and `reasons/api.py:586`:

- restoration from outlist: gated beliefs whose blocker is removed
- outlist-gated beliefs that lose their justification when the blocker goes `IN`

## What “computed as OUT” means for cycle fallback

The key point is that `OUT` is an evaluated truth state, not necessarily an explicit retraction.

For an outlist target:

- if the node is explicitly retracted, it is `OUT`
- if the node has no valid justifications, it is `OUT`
- if the node does not exist, the tests say it is treated as `OUT`
- if the node is computed `OUT` because of cycle fallback, then it is also consumed by outlist checks as `OUT`

The “missing outlist node” semantics are tested in:

```python
tests/test_outlist.py::TestOutlistBasic::test_outlist_node_not_in_network
```

with this expectation:

```text
Outlist node doesn't exist — treated as OUT (absent = not believed).
```

So for cycle fallback specifically: if the evaluator has a recursion/cycle guard that returns `OUT` for an in-progress node, then that node satisfies an `unless`/outlist check because the justification condition says “all outlist nodes must be `OUT`.” In other words, a cycle-fallback `OUT` is not a special “defeated” state; it is the same broad notion of “not currently believed / no valid justification” that outlist checks consume.

Example consequence:

```text
A is cyclic/unfounded -> fallback computes A as OUT
X has SL(antecedents=[], outlist=["A"])
Then X's justification can hold, because A is OUT.
```

That is consistent with “believe X unless A is believed”: if A is not believed — including because it is only cyclic/unfounded — then the `unless A` gate is satisfied.

Caveat: the provided observations do **not** show the actual cycle guard in `reasons/network.py` or wherever the recursive truth-evaluation function lives. They show the outlist condition, the API construction of outlist, and the tests for normal `OUT`/`IN` behavior. To prove exactly how cycles are treated, you need the evaluator function that implements the fallback. In particular, confirm whether cycle fallback returns a normal `"OUT"` string or some separate `"UNKNOWN"` / in-progress state. If it returns normal `"OUT"`, an outlist blocker in a cycle will not block. If it returns an unknown state that is not equal to `"OUT"`, the semantics would differ.

So the precise statement supported by these observations is:

> `SL` with `outlist` implements `unless` as a justification-validity condition: all antecedents `IN` and all outlist nodes `OUT`. Defeat works by putting an `IN` defeater into that outlist. Therefore “computed as OUT” for cycle fallback means “the node has failed to be established `IN` under the current evaluation,” and outlist logic will treat such an `OUT` node as an absent blocker, unless the cycle evaluator uses a distinct unknown state.

## Topics to Explore

- [file] `reasons/network.py` — Contains the actual `Network` evaluation and propagation logic, including the cycle guard; needed to verify whether cycle fallback returns normal `OUT` or a distinct unknown state.
- [function] `reasons/api.py:defeat_justification` — Implements defeasibility by creating a defeater node and appending it to the target justification’s outlist.
- [file] `tests/test_outlist.py` — Encodes intended semantics for absent outlist nodes, multiple blockers, cascading retraction, restoration, and alternate justifications.
- [function] `reasons/network.py:Network.assert_node` — The entry point for making a blocker `IN`, which cascades outlist-gated dependents to `OUT`.
- [general] `justification-evaluation-around-api-1590` — Inspect the exact evaluator using `outlist_ok = all(...)` to see how missing, cyclic, or in-progress nodes are read.

## Beliefs

- `sl-outlist-requires-all-outlist-out` — An `SL` justification holds only when every antecedent is `IN` and every outlist node is `OUT`.
- `node-in-is-any-justification` — A node is `IN` if at least one of its justifications holds; it is `OUT` only when none holds.
- `unless-api-maps-to-outlist` — The API’s `unless` parameter is parsed into `Justification.outlist`, and `unless` without `sl` creates an empty-antecedent `SL` default.
- `defeat-justification-adds-defeater-to-outlist` — `defeat_justification` implements defeat by creating an `IN` defeater node and appending its ID to the target justification’s `outlist`.
- `missing-outlist-node-counts-as-out` — A nonexistent outlist node is treated as `OUT`, so it does not block the default justification.
