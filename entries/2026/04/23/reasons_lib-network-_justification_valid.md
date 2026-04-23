# Function: _justification_valid in reasons_lib/network.py

**Date:** 2026-04-23
**Time:** 16:35

## `_justification_valid` — Justification Validity Check

### Purpose

This is the core predicate of the Truth Maintenance System. It answers one question: **given the current state of the network, does this justification support its conclusion?**

Every truth value computation in the network bottoms out here. When `_compute_truth` needs to decide whether a node should be IN or OUT, it calls `_justification_valid` on each of the node's justifications. If *any* justification is valid, the node is IN. This method defines what "valid" means.

It implements the validity condition from Doyle's TMS: a **support-list (SL) justification** is valid when all its positive dependencies hold and none of its negative dependencies hold. This is the mechanism that enables non-monotonic reasoning — the ability to believe something *unless* contrary evidence appears.

### Contract

**Preconditions:**
- `j` is a `Justification` instance with populated `type`, `antecedents`, and `outlist` fields.
- `self.nodes` reflects the current network state (truth values are up-to-date for the nodes referenced by `j`).

**Postconditions:**
- Returns `True` if and only if the justification is currently satisfied.
- No mutations — this is a pure query against current network state.

**Invariant:** A justification's validity can change whenever any node in its `antecedents` or `outlist` changes truth value. The caller (`_compute_truth` → `_propagate`) is responsible for re-evaluating after such changes.

### Parameters

| Parameter | Type | Meaning |
|-----------|------|---------|
| `j` | `Justification` | The justification to evaluate. Has fields `type` (str), `antecedents` (list of node IDs), and `outlist` (list of node IDs). |

**Edge cases:**
- Empty `antecedents`: `all(...)` over an empty iterable is `True`, so an SL justification with no antecedents is valid as long as the outlist condition holds. This is intentional — it models premises that are "IN unless Y."
- Empty `outlist`: Similarly `True` by vacuous truth. A justification with no outlist is valid whenever all antecedents are IN — pure monotonic support.
- Both empty: Always valid. Used when converting premises to justified nodes (see `challenge` and `supersede`).

### Return Value

- `True` — the justification currently supports its conclusion being IN.
- `False` — the justification does not support its conclusion. Either an antecedent is OUT/missing, an outlist node is IN, or the justification type is unrecognized.

The caller (`_compute_truth`) checks *all* justifications on a node. A single valid justification is sufficient for the node to be IN.

### Algorithm

```
1. Check if the justification type is "SL" or "CP"
   → If neither, return False (unknown type = never valid)

2. Check the inlist (antecedents):
   For each antecedent ID:
     - It must exist in self.nodes  (missing = fails)
     - Its truth_value must be "IN"
   All must pass (conjunction).

3. Check the outlist:
   For each outlist ID:
     - If it does NOT exist in self.nodes → OK (absent = no threat)
     - If it DOES exist → its truth_value must be "OUT"
   All must pass (conjunction).

4. Return inlist_ok AND outlist_ok
```

The **asymmetry** between inlist and outlist handling is critical:
- A **missing antecedent** fails the check (you can't believe X based on evidence that doesn't exist).
- A **missing outlist node** passes the check (the threat you're guarding against doesn't exist yet, so the guard isn't triggered).

This asymmetry is what makes the outlist mechanism work for "believe X unless Y" — Y might not have been added to the network yet, and that's fine. X should be believed until Y actually shows up.

### Side Effects

None. This is a pure function — it reads `self.nodes` but mutates nothing. No logging, no I/O, no state changes.

### Error Handling

No exceptions are raised. Missing nodes are handled inline:
- Missing antecedent → `a in self.nodes` evaluates to `False`, short-circuits to invalid.
- Missing outlist node → `o not in self.nodes` evaluates to `True`, passes.

This silent handling is deliberate — during network construction, nodes may reference IDs that haven't been added yet.

### Usage Patterns

Called from two places:

1. **`_compute_truth`** (`network.py:420`) — iterates over a node's justifications, returns "IN" on the first valid one. This is the primary call site.

2. **`explain`** (`network.py:356`) — when tracing why a node is IN, finds the specific valid justification to report.

Both callers iterate over justifications, so this method is called once per justification per truth value evaluation. During a propagation cascade, it may be called many times as dependent nodes are recomputed.

### Dependencies

- `Justification` dataclass (from `reasons_lib/__init__.py`) — expects `.type`, `.antecedents`, `.outlist` fields.
- `Node` dataclass (from `reasons_lib/__init__.py`) — reads `.truth_value` from `self.nodes`.
- No external dependencies.

### Assumptions Not Enforced by Types

1. **`j.type` is expected to be `"SL"` or `"CP"`**, but the type annotation is just `str`. Any other value silently returns `False`, which could mask typos.
2. **Antecedent and outlist IDs are assumed to be valid node ID strings**, but there's no validation that they correspond to real or future nodes.
3. **`truth_value` is assumed to be either `"IN"` or `"OUT"`** — the string comparison would silently fail on any other value (e.g., `None`, `"UNKNOWN"`).
4. **CP (conditional-proof) justifications are evaluated identically to SL**, but Doyle's original TMS treats them differently. The code handles them the same way, which may be intentional simplification or an incomplete implementation.

---

## Topics to Explore

- [function] `reasons_lib/network.py:_compute_truth` — The immediate caller; understand how multiple justifications combine (disjunctive: any-valid-wins)
- [function] `reasons_lib/network.py:_propagate` — How truth value changes cascade through the dependency graph via BFS
- [function] `reasons_lib/network.py:supersede` — Key user of the outlist mechanism: makes old beliefs go OUT when replacements arrive
- [general] `doyle-tms-sl-justifications` — Doyle's 1979 paper on Support-List justifications and non-monotonic reasoning, the theoretical foundation for this code
- [file] `tests/test_network.py` — Test cases that exercise the inlist/outlist asymmetry and edge cases like empty lists

---

## Beliefs

- `sl-outlist-asymmetry` — Missing antecedents invalidate a justification, but missing outlist nodes do not; this asymmetry enables "believe X unless Y" where Y may not yet exist in the network
- `cp-equals-sl` — CP justifications are evaluated with the exact same logic as SL justifications, despite being a distinct type in Doyle's TMS
- `justification-valid-is-pure` — `_justification_valid` is a pure query with no side effects, no logging, and no mutations to network state
- `unknown-type-returns-false` — Any justification type other than "SL" or "CP" silently returns False, which could mask configuration errors
- `empty-antecedents-vacuously-valid` — An SL justification with an empty antecedent list is valid (vacuous truth), allowing outlist-only justifications to function as "IN unless Y"

