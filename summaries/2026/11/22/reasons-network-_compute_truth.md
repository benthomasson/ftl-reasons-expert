---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: _compute_truth in reasons/network.py

### Purpose

`_compute_truth` is the core truth-value calculation for a justified node in the TMS network. It implements the rule:

> A node is `IN` if at least one of its justifications is currently valid.

It also records which justification was selected as the node’s designated support by setting `node.supporting_justification` to the index of the newest valid justification.

The docstring says “newest” because justifications are appended to `node.justifications`; the method scans the list backwards, so the last-added valid justification wins.

---

### Contract

**Preconditions**

- `node` is expected to be a `Node`-like object with:
  - `node.justifications`
  - `node.truth_value`
  - `node.supporting_justification`
- If `node` has justifications, their antecedents/outlist references are expected to have current `truth_value`s in `self.nodes`.
- The caller is generally expected to pass a node that belongs to this network, though `_compute_truth` itself does not verify that `node` is present in `self.nodes`.

**Postconditions**

For a node with at least one justification:

- If any justification is valid:
  - returns `"IN"`
  - sets `node.supporting_justification` to the index of the newest valid justification.
- If no justification is valid:
  - returns `"OUT"`
  - sets `node.supporting_justification = None`.

For a node with no justifications:

- returns `node.truth_value` unchanged
- does **not** modify `node.supporting_justification`.

**Invariants / assumptions**

- For justified nodes, `supporting_justification` is either `None` or an integer index into `node.justifications`.
- When `_compute_truth` returns `"IN"` for a justified node, the selected `supporting_justification` was valid at the time of the call.
- The method does not assign `node.truth_value`; callers are responsible for storing the returned value.
- The method does not inspect `node.metadata["_retracted"]`; retraction filtering is handled by callers such as `_propagate` and `recompute_all`.
- The method assumes dependency truth values are already current enough for `_justification_valid` to make the right decision.

---

### Parameters

- `node: Node`
  - The node whose truth value should be computed from its justifications.
  - May be a premise (`node.justifications == []`) or a justified/derived node.
  - Edge cases:
    - Empty justifications: returns the current truth value.
    - Multiple valid justifications: selects the highest-index valid one.
    - Invalid/missing antecedents or outlist nodes: delegated to `_justification_valid`.
    - Unknown justification types: `_justification_valid` returns `False`.
    - Retracted nodes: not checked here; caller must decide whether to call this.

---

### Return Value

Returns a string:

- `"IN"` if a justified node has at least one valid justification.
- `"OUT"` if a justified node has no valid justifications.
- `node.truth_value` if the node has no justifications.

The caller must handle the premise case specially: `_compute_truth` does not force premises back to `"IN"`. A retracted premise with no justifications returns its current `"OUT"` value if called directly.

The caller is also responsible for assigning the result to `node.truth_value` when appropriate and propagating any change to dependents.

---

### Algorithm

1. If `node.justifications` is empty:
   - return `node.truth_value`
   - leave `node.supporting_justification` untouched.

2. Iterate over justification indices from newest to oldest:
   ```python
   for i in range(len(node.justifications) - 1, -1, -1):
   ```

3. For each justification, call `self._justification_valid(node.justifications[i])`.

4. On the first valid justification found:
   - set `node.supporting_justification = i`
   - return `"IN"`.

5. If the loop finishes without finding a valid justification:
   - set `node.supporting_justification = None`
   - return `"OUT"`.

This means the method stops at the newest valid justification and does not evaluate older justifications after finding one.

---

### Side Effects

- Mutates `node.supporting_justification` for justified nodes:
  - sets it to an integer index when a valid justification is found
  - sets it to `None` when all justifications are invalid.
- Reads `node.justifications`.
- Reads current truth values of other nodes through `_justification_valid`, which accesses `self.nodes`.
- Does **not** mutate `node.truth_value`.
- Does **not** mutate `node.justifications`.
- Does **not** log anything.
- Does **not** propagate changes to dependents.

---

### Error Handling

`_compute_truth` has no explicit exception handling and raises no deliberate exceptions.

Possible failures come from malformed inputs:

- `AttributeError` if `node` lacks expected attributes.
- `TypeError` if `node.justifications` is not iterable/indexable as expected.
- Errors inside `_justification_valid` if malformed justification objects are passed.

Missing referenced nodes do not necessarily raise:

- `_justification_valid` treats a missing antecedent as not `IN`, so the justification is invalid.
- `_justification_valid` treats a missing outlist node as acceptable/`OUT`, so it does not invalidate the justification.

Unknown justification types are treated as invalid by `_justification_valid`, not as errors.

---

### Usage Patterns

`_compute_truth` is called whenever the network needs to recalculate a node based on its justifications, for example:

- during initial `add_node` when justifications are present
- during `_propagate` when a dependency changes
- during `recompute_all` fixpoint recomputation
- in `add_justification`
- in `remove_justification`
- in `supersede`
- in `challenge`

Typical caller pattern:

```python
old_value = node.truth_value
new_value = self._compute_truth(node)
if old_value != new_value:
    node.truth_value = new_value
    self._propagate(node.id)
```

Caller obligations:

- Ensure relevant antecedent/outlist truth values are up to date.
- Assign the returned value to `node.truth_value` if needed.
- Propagate if the value changed.
- Do not expect `_compute_truth` to restore retracted premises automatically.

`supporting_justification` is consumed by `explain`, which revalidates the selected justification before trusting it and falls back to searching for another valid justification if the stored index is stale.

---

### Dependencies

Internal dependencies:

- `Node` — supplies `justifications`, `truth_value`, and `supporting_justification`.
- `Justification` — supplies `type`, `antecedents`, and `outlist`.
- `self.nodes` — used by `_justification_valid` to resolve antecedent and outlist truth values.
- `self._justification_valid` — implements SL/CP validity semantics.

No external libraries are used directly by this method.

---

## Topics to Explore

- [function] `reasons/network.py:_justification_valid` — Explains exactly how SL/CP antecedents and outlist nodes determine justification validity.
- [function] `reasons/network.py:_propagate` — Shows how `_compute_truth` is used during dependent BFS propagation.
- [function] `reasons/network.py:recompute_all` — Demonstrates fixpoint recomputation when node insertion order or cycles affect truth values.
- [file] `tests/test_network.py` — Provides expected behavior around truth propagation, retraction, and justification validity.
- [general] `Doyle-TMS` — Background on justification-based truth maintenance systems and non-monotonic “believe unless” reasoning.

## Beliefs

- `compute-truth-prefers-newest-valid-justification` — When multiple justifications are valid, `_compute_truth` selects the highest-index valid justification and stores that index in `supporting_justification`.
- `compute-truth-returns-in-for-any-valid-justification` — A justified node returns `"IN"` as soon as `_justification_valid` returns true for any justification.
- `compute-truth-preserves-premise-truth-value` — For a node with no justifications, `_compute_truth` returns the node’s existing `truth_value` and does not modify `supporting_justification`.
- `compute-truth-does-not-assign-truth-value` — `_compute_truth` only returns `"IN"` or `"OUT"`; assigning the result to `node.truth_value` and propagating is the caller’s responsibility.
- `compute-truth-ignores-retraction-metadata` — `_compute_truth` does not inspect `_retracted`; retraction filtering is handled by callers like `_propagate` and `recompute_all`.
