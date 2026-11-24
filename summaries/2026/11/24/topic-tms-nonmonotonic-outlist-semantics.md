---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: The interplay of justification `antecedents` (IN to satisfy) vs `outlist` (OUT to satisfy) explains why *both* are treated as "dependents" here.

## Why both `antecedents` and `outlist` are "dependents"

### The truth-value contract

`reasons/__init__.py:11-12` states the satisfaction rule for SL justifications:

> - SL (Support List): node is IN iff ALL antecedents (inlist) are IN
> - AND ALL outlist nodes are OUT

And the two fields live side by side on the same object (`reasons/__init__.py:20-21`):

```python
antecedents: list[str] = field(default_factory=list)  # inlist: must be IN
outlist: list[str] = field(default_factory=list)      # must be OUT
```

So a justification's validity is a *joint* predicate over both lists — it is neither satisfied nor violated without evaluating both.

### What `dependents` actually is

`reasons/__init__.py:38` defines it as a *reverse index*:

```python
dependents: set[str] = field(default_factory=set)  # reverse index
```

It answers the propagation question: **"if this node flips truth value, whose justifications need re-checking?"** `_propagate` walks it directly — `reasons/api.py:745`:

```python
for dep_id in current.dependents:
```

This is why the design must include outlist nodes in the reverse index. Whether a node is an antecedent of a dependent or appears in that dependent's outlist, **any truth change on that node can invalidate or restore the dependent**:

- `a` (antecedent) flips IN → OUT ⟹ dependent loses a required IN.
- `y` (outlist) flips OUT → IN ⟹ dependent's blocker now fires; justification fails.
- The reverse moves (antecedent OUT → IN, outlist IN → OUT) can *restore* the dependent.

The IN/OUT directional meaning differs by field, but the *causation direction* is the same: the outlist node's state drives the dependent's evaluation just as much as an antecedent's state does. That is the sense in which both are "dependents".

### Concrete evidence in code and tests

**Construction path (add node / add justification).** `reasons/api.py:2579-2582` and `reasons/api.py:2650-2653` add the same `nid` to the `dependents` of both the antecedent node **and** the outlist node:

```python
net.nodes[ant_id].dependents.add(nid)
...
net.nodes[out_id].dependents.add(nid)
```

**Rewrite path.** `reasons/api.py:4925-4933` (`_rewrite_dependents`) is documented as: *"Updates both the justification antecedents/outlist and the dependents"* — reinforcing that the reverse index mirrors the forward justification, covering both lists.

**Rebuild path.** `tests/test_dependents_integrity.py` verifies the invariant explicitly:

- `test_outlist_dependency` — after `_rebuild_dependents()`, `"b" in net.nodes["a"].dependents`, where `b`'s justification has `outlist=["a"]`.
- `test_both_antecedent_and_outlist` — a node with `antecedents=["a"], outlist=["b"]` shows up in **both** `a.dependents` and `b.dependents`.

**Propagation behavior.** `tests/test_outlist.py` demonstrates that cascades ride the exact same `dependents` edges as antecedents:

- `test_assert_outlist_node_retracts_dependent`: asserting `y` (an outlist node) sends `x` OUT — proof that propagation re-evaluated `x` because `y` was in `x`'s *reverse* path.
- `test_retract_outlist_node_restores_dependent`: retracting `y` brings `x` back IN.
- `test_outlist_cascade_through_chain`: asserting `y` cascades through `x.dependents` and on to `z`.

None of this works if the reverse index only covered antecedents — that is the design constraint the observations point to.

### Why the coupling isn't accidental

The evaluation logic that decides IN/OUT is symmetric in its *inputs* even though the predicates differ:

- `reasons/api.py:499-504` inspects `j.antecedents` for SL satisfaction.
- `reasons/api.py:1590-1592` computes `outlist_ok = all(...)` over `j.outlist`.

Both branches feed the same node truth recomputation, which in turn triggers the same `_propagate` loop over `dependents`. So if only antecedents were registered as dependents, a violated outlist would never fire a propagation step and non-monotonic behavior of `unless` would silently break.

### Summary

`dependents` is not "the set of nodes that require this node to be IN" — it is "**the set of nodes whose justification validity is a function of this node's truth value**." Because an SL justification's validity is a function of both its `antecedents` (must be IN) **and** its `outlist` (must be OUT), both kinds of IDs must be recorded as reverse edges. `_propagate` then treats them identically when deciding which nodes to re-evaluate.

## Topics to Explore

- [function] `reasons/api.py:_propagate` — The worklist loop that iterates `current.dependents` (line 745); the single place where the "both are dependents" design actually pays off during truth propagation.
- [function] `reasons/api.py:_rebuild_dependents` — Canonical reverse-index rebuild; the symmetry you see in `test_both_antecedent_and_outlist` originates here.
- [file] `reasons/api.py` — Around lines 490-520 and 1580-1600 for the joint `antecedents`+`outlist` satisfaction logic that makes both membership sets need propagation.
- [function] `reasons/api.py:_rewrite_dependents` — Lines ~4925-4940; the docstring literally ties antecedent/outlist rewrite to dependents rewrite, which is the invariant you want to keep an eye on during node-rename refactors.
- [file] `tests/test_outlist.py` — `TestOutlistCascading`; concrete behavioral proof that outlist-driven flips propagate exactly like antecedent-driven flips.

## Beliefs

- `dependents-is-a-reverse-index` — `Node.dependents` (reasons/__init__.py:38) stores the set of node IDs whose justification validity depends on this node, not the forward edges.
- `sl-requires-antecedents-in-and-outlist-out` — An SL justification is satisfied iff every `antecedents` entry is IN and every `outlist` entry is OUT (reasons/__init__.py:11-12).
- `outlist-nodes-are-propagated-as-dependents` — Both `add_node`/`add_justification` (reasons/api.py:2579-2582, 2650-2653) and `_rebuild_dependents` register the derived node under both antecedent and outlist node `dependents` sets.
- `propagation-iterates-dependents-uniformly` — `_propagate` (reasons/api.py:745) does not distinguish antecedent vs. outlist edges; both trigger re-evaluation through the same `current.dependents` loop.
- `outlist-flips-cascade-through-dependents` — Asserting an outlist node retracts its dependent and retracting it restores the dependent (tests/test_outlist.py:TestOutlistCascading), which is only possible because outlist nodes are in the reverse index.
