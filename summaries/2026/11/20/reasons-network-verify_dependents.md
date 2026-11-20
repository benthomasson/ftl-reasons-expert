---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: verify_dependents in reasons/network.py

## Purpose

`verify_dependents` is a consistency checker for the network’s denormalized reverse dependency index.

Each `Node` stores a `dependents: set[str]` field: the IDs of nodes whose truth values may need recomputation when this node changes. That index is derived from justifications. For every node `N` and every justification of `N`:

- every antecedent of `N` should list `N` as a dependent, because `N` depends on that antecedent being `IN`;
- every outlist node of `N` should also list `N` as a dependent, because `N` depends on that node being `OUT`.

`verify_dependents` reconstructs the expected reverse index from justifications and compares it against the live `node.dependents` sets. It exists to detect drift/corruption in that cache, especially after graph mutations, imports, or manual repairs.

It is a read-only validator. It does not repair anything.

---

## Contract

### Preconditions

- `self.nodes` is a `dict[str, Node]`.
- Each `Node` has:
  - `.id`
  - `.justifications`
  - `.dependents`
- Each justification has:
  - `.antecedents`
  - `.outlist`
- `node.dependents` behaves like a `set[str]`, because the function uses set subtraction:
  ```python
  live - exp
  exp - live
  ```
- The graph is assumed to be stable during verification; mutating `self.nodes` while iterating is not supported.

### Postconditions

- Returns `[]` if, for every existing node, its live `dependents` set exactly matches the set implied by all existing nodes’ justifications.
- Returns one or more human-readable error strings if any existing node has:
  - extra dependents not implied by any justification;
  - missing dependents implied by some justification.
- The network is not modified.

### Core invariant checked

For every existing node `D` and every justification `J` on `D`:

- for every `A in J.antecedents` where `A in self.nodes`, `D.id` must be in `self.nodes[A].dependents`;
- for every `O in J.outlist` where `O in self.nodes`, `D.id` must be in `self.nodes[O].dependents`.

Conversely, no existing node’s `dependents` set should contain IDs that are not implied by that rule.

---

## Parameters

This method has no explicit parameters beyond `self`.

- `self`: a `Network` instance.
  - Expected to have already-populated `self.nodes`.
  - Works correctly on an empty network and returns `[]`.

There are no caller-supplied node IDs, sets, or flags.

---

## Return Value

Returns `list[str]`.

- `[]` means the reverse dependency index is consistent with the justifications.
- Non-empty list means inconsistencies were found.
- Each error string follows one of these forms:
  - `f"{nid}: extra dependents {extras}"`
  - `f"{nid}: missing dependents {missing}"`

A single node may produce both an “extra dependents” error and a “missing dependents” error.

The set representation in the message is Python’s default set formatting, so callers should not depend on stable ordering of IDs inside `{...}`.

---

## Algorithm

Step by step:

1. Build an empty expected reverse index:
   ```python
   expected = {nid: set() for nid in self.nodes}
   ```
   This creates an expected dependent set for every currently known node.

2. Walk every node in the network:
   ```python
   for node in self.nodes.values():
   ```
   Treat `node.id` as the potential dependent.

3. For each justification on that node:
   - For each antecedent ID:
     ```python
     if ant_id in self.nodes:
         expected[ant_id].add(node.id)
     ```
     The antecedent’s dependents should include the justifying node.
   - For each outlist ID:
     ```python
     if out_id in self.nodes:
         expected[out_id].add(node.id)
     ```
     Outlist nodes are negative dependencies: if they become `IN`, the justified node may go `OUT`. Therefore they must also know the justified node as a dependent.

4. Compare expected against live state for every node:
   ```python
   live = self.nodes[nid].dependents
   exp = expected.get(nid, set())
   extras = live - exp
   missing = exp - live
   ```
   - `extras`: live dependents not justified by any justification.
   - `missing`: justified dependents absent from the live index.

5. Append formatted error strings for non-empty `extras` or `missing`.

6. Return the collected errors.

---

## Side Effects

None.

- Does not mutate `self.nodes`.
- Does not mutate `node.dependents`.
- Does not rebuild anything.
- Does not write to `self.log`.
- Does not perform I/O.
- Only creates local `expected` and `errors` objects and temporary set differences.

---

## Error Handling

The function does not raise domain-specific errors for inconsistency. Inconsistencies are returned as strings.

Potential Python exceptions can still occur if structural assumptions are violated:

- `AttributeError` if a node lacks `.dependents`, or a justification lacks `.antecedents`/`.outlist`.
- `TypeError` if `.dependents` is not set-like and does not support subtraction with a `set`.
- Mutation during iteration could cause runtime behavior problems, though no explicit guard exists.

Important: missing referenced nodes are silently skipped during expected-index construction:

```python
if ant_id in self.nodes:
    expected[ant_id].add(node.id)
```

So if a justification references a non-existent antecedent or outlist node, `verify_dependents` does not report that dangling reference. It only validates reverse-index entries for nodes that currently exist.

If a live `dependents` set contains an unknown ID, that unknown ID will appear in `extras`, because it cannot be in the expected set.

---

## Usage Patterns

Typical uses:

- After importing or deserializing a network, verify that the reverse index was rebuilt correctly.
- In tests such as `tests/test_dependents_integrity.py`.
- After mutations like `add_node`, `add_justification`, `remove_justification`, `challenge`, `supersede`, or `convert_to_premise`.
- As a diagnostic/audit helper before serializing or asserting graph integrity.

Typical caller pattern:

```python
errors = network.verify_dependents()
assert not errors, "\n".join(errors)
```

If errors are present, the caller must decide how to repair them. The canonical repair function is likely `_rebuild_dependents()`, which recomputes the entire reverse index from justifications.

Caller obligations:

- Do not treat `verify_dependents` as a repair mechanism.
- Do not expect it to validate truth values, nogoods, Merkle hashes, or missing justification references.
- Ensure graph mutations have finished before calling it.

---

## Dependencies

Internal structural dependencies:

- `Network.nodes`
- `Node.id`
- `Node.justifications`
- `Node.dependents`
- `Justification.antecedents`
- `Justification.outlist`

No external modules are used directly. The function relies only on Python built-ins (`dict`, `set`, iteration, set subtraction).

---

## Assumptions Not Enforced by the Type System

- `node.dependents` is assumed to be a mutable/set-like object of strings, not a list or tuple.
- All node IDs are assumed to be strings.
- Justification antecedent and outlist entries are assumed to be iterable collections of node ID strings.
- The graph is assumed not to contain meaningful dangling justification references, because this function skips them.
- The reverse index is assumed to be derivable entirely from `justifications`. Any manually maintained dependency relation not represented in a justification will be reported as an “extra dependent.”
- The function assumes no concurrent mutation of `self.nodes` during execution.

---

## Topics to Explore

- [function] `reasons/network.py:_rebuild_dependents` — Canonical writer of the reverse index; compare its construction logic with `verify_dependents`.
- [file] `tests/test_dependents_integrity.py` — Shows expected behavior for extra dependents, missing dependents, and repair flows.
- [function] `reasons/network.py:_propagate` — Consumes `node.dependents`; broken reverse-index entries directly affect truth-value propagation.
- [function] `reasons/network.py:add_justification` — One of the mutation points that updates the reverse index and must keep it consistent.
- [general] `dependents-index-maintenance` — Explore every code path that adds, removes, rebuilds, or manually adjusts `dependents` sets.

---

## Beliefs

- `verify-dependents-reads-only` — `verify_dependents` does not mutate the network, nodes, dependents sets, or audit log.
- `expected-dependents-are-derived-from-antecedents-and-outlists` — Both `Justification.antecedents` and `Justification.outlist` contribute expected entries to the reverse index.
- `verify-dependents-ignores-missing-node-references` — Justification references to IDs not present in `self.nodes` are silently skipped during expected-index construction.
- `verify-dependents-returns-errors-not-exceptions` — Extra or missing dependents are reported in the returned list rather than raised as exceptions.
- `verify-dependents-assumes-set-valued-dependents` — The implementation requires `Node.dependents` to support set subtraction with another set.
