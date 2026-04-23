# Topic: Non-monotonic reasoning: how outlists enable "believe X unless Y", supersession, and challenges

**Date:** 2026-04-23
**Time:** 16:44

# Non-Monotonic Reasoning: Outlists, Supersession, and Challenges

## What "non-monotonic" means here

In classical logic, adding new information never invalidates old conclusions — that's monotonic reasoning. Real-world reasoning doesn't work that way. Learning "the road is flooded" should retract "take route A" even though all the evidence that originally supported route A is still true.

Doyle's TMS solves this with **SL (support-list) justifications** that have two parts:

- **Inlist** (antecedents): nodes that must be IN
- **Outlist**: nodes that must be OUT

A justification is valid only when *all* antecedents are IN **and** *all* outlist nodes are OUT. This is the core of "believe X unless Y" — X has a justification with Y in its outlist.

## The validation logic

The implementation lives in `reasons_lib/network.py:_justification_valid()` (around line 717):

```python
inlist_ok = all(
    a in self.nodes and self.nodes[a].truth_value == "IN"
    for a in j.antecedents
)
outlist_ok = all(
    o not in self.nodes or self.nodes[o].truth_value == "OUT"
    for o in j.outlist
)
return inlist_ok and outlist_ok
```

Notice the asymmetry: absent antecedents fail validation (missing evidence = not supported), but absent outlist nodes pass (missing counter-evidence = no objection). This is tested explicitly at `tests/test_outlist.py:65-69` — a nonexistent outlist node is treated as OUT, so the belief holds.

A node is IN if **any** of its justifications is valid (`_compute_truth()`, around line 704). This means a node can survive an outlist violation if it has a backup justification, tested at `tests/test_outlist.py:133-146`:

```python
# X has two justifications: SL(unless Y) and SL(A)
# Assert Y — first justification fails, but second keeps X IN
net.assert_node("y")
assert net.nodes["x"].truth_value == "IN"
```

## Cascading through the dependency graph

When an outlist node's status changes, propagation follows. Outlist nodes are registered in the `dependents` set during `add_node()` (line ~60) and `add_justification()` (line ~352), so the BFS propagation in `_propagate()` (line ~675) finds them.

The cascade chain is tested at `tests/test_outlist.py:99-110`:

```python
# X unless Y, Z depends on X. Assert Y → both X and Z go OUT.
changed = net.assert_node("y")
assert net.nodes["x"].truth_value == "OUT"
assert net.nodes["z"].truth_value == "OUT"
```

Crucially, this is **reversible** — retracting Y brings X back IN, which cascades Z back IN too (`tests/test_outlist.py:86-93`).

## Supersession: one belief replacing another

`Network.supersede()` (around line 287) is built entirely on the outlist mechanism. It adds the new node's ID to the old node's outlist:

```python
for j in old_node.justifications:
    if new_id not in j.outlist:
        j.outlist.append(new_id)
```

While the new belief is IN, the old one is automatically OUT. If the new belief is later retracted, the old one comes back — supersession is reversible by design, unlike deletion.

## Challenges and defenses: dialectical argumentation

The challenge/defend pattern (`tests/test_dialectical.py`) is a direct application of outlists to argumentation.

**Challenge** (`Network.challenge()`, around line 374): Creates a new premise node (the challenge) and adds it to the target's outlist. Since the challenge starts IN, the target immediately goes OUT. If the target is a bare premise with no justifications, it gets converted to a justified node with `SL(antecedents=[], outlist=[challenge_id])`.

**Defend** (`Network.defend()`, around line 449): Defense is implemented as *challenging the challenge*. It calls `self.challenge(challenge_id, reason)` internally. The defense node goes into the challenge's outlist, making the challenge go OUT, which restores the original target.

This creates dialectical chains that can go arbitrarily deep (`tests/test_dialectical.py:163-187`):

```
A (premise) → challenged → A goes OUT
  → defended → challenge goes OUT → A restored IN
    → defense challenged → defense OUT → challenge restored → A OUT again
      → defense defended → counter-challenge OUT → defense IN → challenge OUT → A IN
```

Each level uses the same outlist mechanism recursively. The entire chain is maintained by the propagation engine — no special-case code.

## Persistence

Outlists survive SQLite round-trips. The `justifications` table stores `outlist_json` (`reasons_lib/storage.py:31`), and on load, the dependent index is rebuilt for both antecedents and outlist nodes (`storage.py:158-162`). The test at `tests/test_outlist.py:177-213` verifies that outlist propagation works correctly after a save/load cycle.

## The hard parts

**Multiple outlist nodes act as conjunction**: all must be OUT. `tests/test_outlist.py:116-131` — "X unless Y or Z" requires both Y and Z to be OUT. Asserting just one is enough to defeat the justification.

**Agent namespacing**: When beliefs are imported from external agents, outlist references get namespaced (e.g., `blocker` → `agent-name:blocker`). The import tests at `tests/test_import_agent.py:196-226` verify that outlist relationships survive namespacing.

**Dependency-directed backtracking** (`tests/test_backtracking.py`): When a contradiction (nogood) is discovered, the system traces backward through justifications to find which premises are responsible, then retracts the one with minimal disruption. This works alongside outlists — backtracking deals with contradictions, while outlists deal with defeasibility.

## Topics to Explore

- [function] `reasons_lib/network.py:supersede` — How reversible belief replacement works, including metadata tracking and edge cases around premise vs. justified nodes
- [file] `reasons_lib/derive.py` — The GATE belief pattern that uses outlists to connect positive claims with negative conditions (bugs/gaps), generating higher-level derived beliefs
- [file] `tests/test_sync_agent.py` — How outlist-gated beliefs (especially `:inactive` kill switches) survive multi-agent sync, import, and re-sync cycles
- [function] `reasons_lib/network.py:_propagate` — The BFS propagation engine that makes cascading work for both antecedent and outlist changes
- [general] `entrenchment-and-backtracking` — How `find_culprits` scores premises by dependent count and entrenchment metadata to choose minimal-disruption retractions

## Beliefs

- `outlist-absent-means-out` — An outlist node that doesn't exist in the network is treated as OUT, meaning the justification is satisfied; this allows forward references and missing counter-evidence to be permissive
- `challenge-is-outlist-injection` — The challenge mechanism works by adding a new premise node to the target's outlist, not by modifying the target's truth value directly; all status changes flow through normal propagation
- `defend-is-recursive-challenge` — Defense against a challenge is implemented by calling `challenge()` on the challenge node itself, using the same outlist mechanism recursively
- `supersession-is-reversible` — `supersede()` adds the new node to the old node's outlist rather than deleting the old node; retracting the new node automatically restores the old one
- `multiple-outlist-is-conjunction` — When a justification has multiple outlist entries, ALL must be OUT for the justification to be valid; any single outlist node going IN defeats the entire justification

