---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: _justification_valid in reasons/network.py

# `_justification_valid` — Explanation

## Purpose

`_justification_valid` is the **ground truth predicate for the truth-maintenance system (TMS)**. It answers a single question: given the current truth values of all nodes in the network, does this particular justification *currently hold*?

A node's truth value is derived entirely from this predicate: `_compute_truth` walks a node's justifications and returns `"IN"` if *any* of them passes this check, `"OUT"` if none do. So this method is the mechanism that implements Doyle's "believe X unless Y" non-monotonic semantics: a justification is valid only when all its **antecedents** are IN *and* none of its **outlist** entries are IN.

## Contract

**Preconditions**
- `j` is a `Justification` instance.
- `self.nodes` is the authoritative live map of node IDs → `Node`.
- No guarantee that every ID referenced in `j.antecedents` or `j.outlist` is present in `self.nodes` (dangling references are tolerated — see below).

**Postconditions**
- Returns `True` iff:
  1. `j.type` is `"SL"` or `"CP"`, **and**
  2. every antecedent `a` satisfies `a in self.nodes and self.nodes[a].truth_value == "IN"`, **and**
  3. every outlist entry `o` satisfies `o not in self.nodes or self.nodes[o].truth_value == "OUT"`.
- No state is mutated.

**Invariants upheld**
- Pure function: no reads outside `self.nodes` / `j`, no writes anywhere.
- Deterministic for a fixed `(self.nodes, j)` snapshot.

**Invariants *not* enforced (type-system gaps)**
- `j.type` is a `str`; the code assumes it is one of `{"SL", "CP", ...}`. Unknown types silently return `False`.
- `Node.truth_value` is a `str`; only `"IN"` and `"OUT"` have meaning here. Any other string is treated as "not IN" for antecedents and "not OUT" for outlist entries (so a third state would be interpreted as OUT for antecedent purposes and as neither IN nor OUT for outlist purposes — effectively satisying the outlist check).
- Justification lists are assumed to be iterable; no length or content checks.

## Parameters

| Parameter | Type | Meaning | Edge cases |
|---|---|---|---|
| `j` | `Justification` | The justification whose truth is being evaluated. Requires `.type`, `.antecedents` (iterable of node IDs), `.outlist` (iterable of node IDs). | Empty `antecedents` and empty `outlist` → both `all(...)` calls are vacuous `True` (a good "always valid" justification). `j.type` not in `("SL", "CP")` → short-circuits to `False`. |

## Return Value

Returns `bool`:
- `True` — justification currently holds; the owning node may be `IN` via this justification.
- `False` — justification fails (unknown type, missing/OUT antecedent, or present-and-IN outlist member).

There is no other return channel — no exceptions, no sentinel values. Callers must treat `False` as "this justification does not support the node right now."

## Algorithm

Step by step:

1. **Type gate** — `if j.type in ("SL", "CP")`. Only these two are recognized. Any other type (e.g. an `"AND"`, `"OR"`, or custom type) unconditionally returns `False`. (Note: the docstring only mentions `SL`, but `CP` is handled identically.)
2. **Antecedent check** — Generate `for a in j.antecedents`: each `a` contributes `True` only if `a in self.nodes` **and** `self.nodes[a].truth_value == "IN"`, then reduce with `all(...)`.
   - Missing antecedent ⇒ `False` for that entry ⇒ whole antecedent clause fails. **Antecedents fail-closed on dangling references.**
3. **Outlist check** — Generate `for o in j.outlist`: each `o` contributes `True` if `o not in self.nodes` **or** `self.nodes[o].truth_value == "OUT"`, then reduce with `all(...)`.
   - Missing outlist entry ⇒ `True` for that entry. **Outlist fails-open on dangling references** — an outlist node that isn't loaded yet cannot "kill" the justification.
4. **Conjunction** — `return inlist_ok and outlist_ok`. Both clauses must be satisfied.

The asymmetric dangling-reference semantics are the interesting bit:

| Reference site | Node present? | Node truth | Effect |
|---|---|---|---|
| Antecedent | no | — | **invalidates** justification |
| Antecedent | yes | `"IN"` | contributes True |
| Antecedent | yes | anything else | **invalidates** justification |
| Outlist | no | — | contributes True |
| Outlist | yes | `"IN"` | **invalidates** justification |
| Outlist | yes | anything else | contributes True |

In short: antecedents are "must be IN"; outlist entries are "must not be IN". A dangling reference can never be IN, so an outlist reference to a deleted/unknown node is silently satisfied.

## Side Effects

None. No mutations, no logging, no imports executed at call time, no I/O. Completely pure.

## Error Handling

- **No exceptions raised** by this function directly.
- Missing antecedent IDs are not exceptions — they yield `False`.
- Missing outlist IDs are not exceptions — they yield `True` (per entry).
- Unknown `j.type` values are not exceptions — they yield `False`.
- Because `j.antecedents` and `j.outlist` are assumed iterables, a malformed `Justification` (e.g. `None`) would surface as a `TypeError` from `all(...)` — but that is not defended against by this method.

## Usage Patterns

`_justification_valid` is the "leaf" predicate of a small hierarchy:

- `_compute_truth(node)` — iterates justifications **backwards** (newest first), returns `"IN"` on the first valid one and records `node.supporting_justification`; returns `"OUT"` if none is valid.
- `_propagate(changed_id)` — BFS over `dependents`, calling `_compute_truth` (and therefore `_justification_valid`) at each dependent after every truth-value change.
- `recompute_all()` — fixpoint loop that repeatedly calls `_compute_truth` on every derived node until nothing changes.
- `explain(node_id)` — for an `IN` node, iterates justifications (preferring `supporting_justification`) and only accepts one that returns `True` here; for an `OUT` node it reports which antecedents failed / outlist entries are violated.

Callers do not need to catch exceptions and do not need to worry about side effects. The main caller obligation is: **keep `self.nodes` consistent** — since a dangling antecedent silently disables a justification, `dependents` bookkeeping bugs (`_rebuild_dependents` / `verify_dependents` exist for exactly this reason) will surface as nodes unexpectedly going OUT.

## Dependencies

- `Justification` (from `reasons/__init__.py` via `from . import Node, Justification, Nogood`) — must expose `.type`, `.antecedents`, `.outlist`.
- `Node.truth_value` (from `reasons/__init__.py`) — the field compared against `"IN"` / `"OUT"`.
- No external libraries.

## Topics to Explore

- [function] `reasons/network.py:_compute_truth` — The function that turns this predicate into a `"IN"`/`"OUT"` verdict and records `supporting_justification`.
- [function] `reasons/network.py:_propagate` — How a change in one node's truth can invalidate justifications downstream and trigger BFS recomputation.
- [function] `reasons/network.py:explain` — How the same predicate drives user-facing "why is this IN/OUT" explanations.
- [file] `reasons/__init__.py` — The `Justification` and `Node` dataclasses; the actual shape of `.type`, `.antecedents`, and `.outlist`.
- [general] `doyle-tms-outlist-semantics` — Background on Doyle's Truth Maintenance System and why the inlist/outlist asymmetry exists.

## Beliefs

- `justification-validation-requires-antecedents-in` — A justification is valid only if every antecedent exists in `self.nodes` and has `truth_value == "IN"`.
- `missing-outlist-nodes-satisfy-empty-clause` — An outlist entry referencing a nonexistent node contributes `True` to the outlist check (dangling outlist references do not invalidate justifications).
- `justification-type-filter-rejects-unknown-types` — Any `j.type` other than `"SL"` or `"CP"` causes `_justification_valid` to return `False` unconditionally.
- `justification-validation-is-pure` — `_justification_valid` performs no mutation, logging, or I/O; it only reads `self.nodes`, `j.type`, `j.antecedents`, and `j.outlist`.
- `compute-truth-depends-on-justification-validity` — `_compute_truth` returns `"IN"` iff at least one of a node's justifications passes `_justification_valid`, walking from newest to oldest.
