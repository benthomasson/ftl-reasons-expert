---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: verify_justification in reasons/merkle.py

## Purpose

`verify_justification` validates one stored justification hash against a fresh Merkle recomputation. It exists to detect integrity failures where either the owning node’s text, an antecedent node’s text, or the recursive antecedent chain has changed since the justification’s `content_hash` was recorded.

It is not a general verifier for `node.text_hash`; that is handled by `verify_node`. This function specifically checks whether `node.justifications[j_index].content_hash` still matches the current Merkle root for that justification.

## Contract

### Preconditions

- `node` is expected to expose:
  - `node.id`
  - `node.text`
  - `node.justifications`, a sequence supporting `len()` and integer indexing
- `j_index` is intended to be an integer index into `node.justifications`.
- Each justification is expected to expose:
  - `.content_hash`
  - `.antecedents`, an iterable of node IDs
- `net` is expected to expose `net.nodes`, a mapping-like object from node ID to node-like objects.
- Nodes reachable through antecedents are expected to expose text, justifications, and supporting-justification fields compatible with `_fresh_merkle_hash`.
- If `cache` is supplied, it is trusted as a mapping of node ID to Merkle hash for the current state of `net`.

### Postconditions

- Returns `None` if:
  - `j_index` is negative or out of range.
  - The selected justification has a falsy `content_hash`.
  - The recomputed content hash matches the stored hash.
- Returns a `chain_mutation` finding dict if the stored hash is truthy and differs from the recomputed hash.
- Does not mutate `node`, the justification object, or `net`.
- May mutate `cache` if supplied.

### Invariants

- The recomputation uses current text via `compute_text_hash(node.text)`, not stored `node.text_hash`.
- Antecedent Merkle hashes are recomputed from current text and current supporting justifications, not from stored antecedent hashes.
- Antecedent hashes are sorted before hashing, so antecedent order does not matter.
- Duplicate antecedent IDs are included as many times as they appear, because the generator iterates every antecedent entry.
- Missing antecedent nodes are silently skipped because `_fresh_content_hash` filters with `if a in net.nodes`.
- A supplied cache is trusted blindly; stale cache entries can produce an incorrect verification result.

## Parameters

- `node`
  - The node owning the justification.
  - Must have `.id`, `.text`, and `.justifications`.
  - `node.id` is only used in the returned finding dict.
  - If `node` is malformed or missing attributes, exceptions propagate.

- `j_index`
  - Index into `node.justifications`.
  - Negative values return `None`.
  - Values `>= len(node.justifications)` return `None`.
  - Non-integer values may raise `TypeError` during comparison or indexing.
  - Since `bool` is an `int` subclass in Python, `True`/`False` can act as indices.

- `net`
  - Must provide `net.nodes`, mapping node IDs to node objects.
  - Used recursively to compute antecedent Merkle hashes.
  - Missing antecedent IDs are skipped rather than raising.

- `cache`
  - Optional dict-like object mapping node ID to Merkle hash.
  - If `None`, `compute_content_hash` creates a temporary local cache that is discarded.
  - If provided, recursive Merkle hash computations may populate it.
  - The function assumes the cache reflects the current state of `net`. It does not validate or invalidate stale entries.

## Return Value

Returns either `None` or a finding dictionary.

On mismatch:

```python
{
    "type": "chain_mutation",
    "node_id": node.id,
    "justification_index": j_index,
    "stored_hash": j.content_hash,
    "current_hash": current,
}
```

On clean verification, missing hash, or invalid index:

```python
None
```

Callers must handle the ambiguity of `None`: it means either “clean” or “nothing to verify.” `verify_all` disambiguates by separately checking whether `content_hash` is missing.

## Algorithm

1. Validate `j_index`:
   - If it is less than `0`, return `None`.
   - If it is greater than or equal to `len(node.justifications)`, return `None`.

2. Fetch the justification:
   - `j = node.justifications[j_index]`

3. Skip if no stored hash exists:
   - If `not j.content_hash`, return `None`.

4. Recompute the justification content hash:
   - Call `compute_content_hash(node, j, net, cache)`.
   - That function computes `text_hash = sha256(node.text)`.
   - It recursively computes Merkle hashes for all antecedents of `j` that exist in `net.nodes`.
   - Each antecedent Merkle hash is computed from the antecedent’s current text and its own supporting justification chain.
   - The recursive computation uses a `visiting` set to avoid infinite recursion on cycles; a back-edge returns `sha256("")`.
   - Antecedent hashes are sorted.
   - The final content hash is:
     - `sha256(text_hash)` if there are no antecedent hashes.
     - `sha256(text_hash + "|" + "|".join(sorted_antecedent_hashes))` otherwise.

5. Compare:
   - If `current != j.content_hash`, return the `chain_mutation` finding.
   - Otherwise return `None`.

## Side Effects

- May populate or mutate the supplied `cache` with recursive Merkle hashes.
- Does not mutate the node, justification, or network.
- Does not perform I/O, logging, or external calls.
- If `cache=None`, a temporary local cache is created and discarded.

## Error Handling

Swallowed conditions:

- Invalid `j_index` returns `None`.
- Missing/falsy `content_hash` returns `None`.
- Missing antecedent nodes are silently excluded from recomputation.

Propagated exceptions:

- `AttributeError` for malformed node or justification objects.
- `TypeError` for non-sequence `justifications`, non-integer index operations, or non-string text passed to `.encode()`.
- `IndexError` for invalid `supporting_justification` indices reached during recursive Merkle hashing.
- `KeyError` or custom mapping errors if `net.nodes` behaves inconsistently with its `in` check.
- Stale or malformed cache values are not detected.

The function does not validate hash format, types, or whether the stored hash was originally produced by the same algorithm.

## Usage Patterns

Typical usage is through `verify_all`, which iterates every node and every justification index, passing a shared cache during the audit:

```python
for ji in range(len(node.justifications)):
    f = verify_justification(node, ji, net, cache)
```

Caller obligations:

- Use a fresh cache per audit, or ensure the cache is valid for the current state of `net`.
- Provide a `net` containing all relevant antecedent nodes if the stored hash was computed with them present.
- Do not treat `None` alone as proof that a hash existed and was clean; check `j.content_hash` separately if missing hashes matter.
- Ensure `j_index` is valid if the caller needs to distinguish invalid-index behavior from clean verification.

## Dependencies

Internal dependencies:

- `compute_text_hash`
- `compute_content_hash`
- `_fresh_content_hash`
- `_fresh_merkle_hash`

External dependency:

- `hashlib` via the hashing helpers.

The function is duck-typed. There is no enforced node, justification, or network superclass in this module.

## Topics to Explore

- [file] `tests/test_merkle.py` — Verify expected behavior for mismatches, missing hashes, recursion, and cache use.
- [function] `reasons/merkle.py:compute_content_hash` — Understand the exact justification hash formula and antecedent filtering.
- [function] `reasons/merkle.py:_fresh_merkle_hash` — Inspect recursive Merkle hashing, cycle handling, and cache/visiting semantics.
- [function] `reasons/merkle.py:verify_all` — See how this verifier is orchestrated and how missing hashes are counted.
- [general] `justification-graph-model` — Explore node/justification structures and the role of `supporting_justification`.

## Beliefs

- `verify-justification-returns-none-without-content-hash` — A falsy `content_hash` causes an immediate `None` return before any recomputation.
- `verify-justification-uses-current-text` — The recomputed hash is based on `compute_text_hash(node.text)` and fresh recursive hashes, not stored `text_hash` values.
- `verify-justification-ignores-missing-antecedents` — Antecedent IDs not present in `net.nodes` are silently excluded from the recomputed justification hash.
- `verify-justification-mutates-supplied-cache` — When `cache` is passed, recursive Merkle hashes may be inserted into it, though node and justification objects are not modified.
- `verify-justification-returns-chain-mutation-finding` — A mismatch between stored and recomputed hashes returns a dict with type `"chain_mutation"`, node ID, justification index, and both hashes.
