---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: verify_all in reasons/merkle.py

## Purpose

`verify_all(net)` is the batch integrity-check entry point for a Merkle-hashed justification graph. It walks every node and every justification in `net`, checks stored hashes against hashes recomputed from current text/content, and returns a summary containing:

- a list of mismatch findings, and
- a count of stored hashes that are missing and therefore unverifiable.

It is intended for auditing a loaded network without modifying it. It does not repair or backfill hashes.

## Contract

### Preconditions

- `net` must expose a mapping-like `net.nodes` attribute.
- `net.nodes` must map node IDs to node objects.
- Node objects must expose:
  - `text`
  - `text_hash`
  - `id`
  - `justifications`
- Justification objects must expose:
  - `content_hash`
  - `antecedents`
- `node.justifications` must be a sequence supporting `len()` and integer indexing.
- `net.nodes` keys must be mutually sortable because the function calls `sorted(net.nodes.items())`.
- The network graph should not be mutated while `verify_all` is running, because the shared `cache` assumes stable recomputed Merkle hashes during the call.

### Postconditions

- Returns a dict:
  ```python
  {"findings": list[dict], "missing_hashes": int}
  ```
- Does not mutate `net`, nodes, or justifications.
- `findings` contains only mismatches where a stored hash exists and differs from the recomputed hash.
- `missing_hashes` counts falsy stored hashes among visited node `text_hash` fields and justification `content_hash` fields.
- Every node is checked exactly once.
- Every justification on every node is checked exactly once.
- Traversal order is deterministic because nodes are sorted by ID.

### Invariants

- Missing hashes are not treated as mismatches; they are counted separately.
- A node with a missing `text_hash` can still have its justifications checked, because `compute_content_hash` recomputes the node text hash from `node.text`.
- The `cache` is local to this call and shared across all justification checks.
- `verify_all` does not validate hash string format, length, or algorithm; it only compares values.

## Parameters

### `net`

Expected to be a network/model object whose `.nodes` attribute is a mapping of node ID to node object.

Edge cases:

- `net.nodes == {}` returns `{"findings": [], "missing_hashes": 0}`.
- If node IDs are mixed and non-orderable types, `sorted(net.nodes.items())` raises `TypeError`.
- If a justification references an antecedent ID not present in `net.nodes`, `compute_content_hash` silently ignores that antecedent because `_fresh_content_hash` filters with `if a in net.nodes`.
- If the graph contains cycles, the underlying Merkle computation uses a sentinel `compute_text_hash("")` for back-edges rather than raising a cycle error.

## Return Value

Returns a plain dict with two keys:

```python
{
    "findings": findings,
    "missing_hashes": missing,
}
```

- `findings` is a list of finding dicts produced by `verify_node` and `verify_justification`.
  - `verify_node` findings have type `"text_mutation"`.
  - `verify_justification` findings have type `"chain_mutation"`.
- `missing_hashes` is an integer count of falsy stored hashes encountered.

Important: the docstring says “return list of mismatches,” but the actual return value is a dict. Callers expecting a list will break.

The caller must handle concerns separately:

- `findings` means actual detected mismatches.
- `missing_hashes > 0` means some hashes could not be verified because they are missing.
- The function does not return which specific hashes are missing; it only counts them.

## Algorithm

1. Initialize:
   - `findings = []`
   - `missing = 0`
   - `cache = {}`

2. Iterate over nodes in sorted ID order:
   ```python
   for nid, node in sorted(net.nodes.items()):
   ```
   `nid` is not otherwise used; sorting makes traversal deterministic.

3. For each node, call `verify_node(node)`:
   - If it returns a finding, append it to `findings`.
   - Otherwise, if `node.text_hash` is falsy, increment `missing`.

4. For each justification index on the node:
   ```python
   for ji in range(len(node.justifications)):
   ```
   call:
   ```python
   verify_justification(node, ji, net, cache)
   ```

5. If `verify_justification` returns a finding, append it to `findings`.
   Otherwise, if that justification’s `content_hash` is falsy, increment `missing`.

6. Return:
   ```python
   {"findings": findings, "missing_hashes": missing}
   ```

The shared `cache` is used by `compute_content_hash` through `verify_justification`, allowing recursive antecedent Merkle hashes to be reused across checks during this call.

## Side Effects

- No mutations to `net`, nodes, or justifications.
- Mutates only local variables: `findings`, `missing`, and `cache`.
- The cache is populated during Merkle recomputation via `_fresh_merkle_hash`.
- No file I/O, network I/O, logging, or external service calls.
- Can be CPU-intensive for large graphs because it may recursively recompute Merkle hashes for antecedents.

## Error Handling

`verify_all` has no `try/except`. Exceptions propagate to the caller.

Possible failures include:

- `AttributeError` if `net` lacks `.nodes`, or nodes/justifications lack expected attributes such as `text`, `text_hash`, `justifications`, `content_hash`, or `antecedents`.
- `TypeError` if `net.nodes` keys are not mutually sortable.
- `AttributeError` or `UnicodeEncodeError` if `node.text` is not a string or contains characters UTF-8 cannot encode.
- `IndexError` from lower-level Merkle computation if a node used as an antecedent has an invalid `supporting_justification`.
- `RecursionError` for extremely deep recursive antecedent chains.
- `KeyError` is generally avoided for missing antecedents because `_fresh_content_hash` filters with `if a in net.nodes`, but inconsistent custom mappings could still cause lookup errors.

Missing stored hashes are not errors. They are returned as `missing_hashes`.

Invalid justification indices are not possible from `verify_all` itself because it iterates `range(len(node.justifications))`. `verify_justification` would return `None` for an invalid index if called directly.

## Usage Patterns

Typical usage:

```python
report = verify_all(net)

if report["findings"]:
    # handle actual hash mismatches
    ...

if report["missing_hashes"]:
    # optionally backfill missing hashes and re-run
    backfill_hashes(net)
    report = verify_all(net)
```

Callers should:

- Inspect `findings` for tampering or stale hash data.
- Treat `missing_hashes > 0` as “unverifiable entries exist,” not as “clean.”
- Avoid mutating `net` while the function runs.
- Not expect the return value to identify which hashes are missing.
- Not expect `verify_all` to backfill or repair anything.

## Dependencies

- Internal:
  - `verify_node`
  - `verify_justification`
  - indirectly `compute_content_hash`, `_fresh_content_hash`, and `_fresh_merkle_hash`
- External:
  - `hashlib` is used by lower-level hashing functions.
- Data model assumptions:
  - `net.nodes` mapping
  - node and justification attribute shapes described above

## Assumptions Not Enforced by the Type System

- `net.nodes` keys are sortable.
- `node.id` exists whenever a mismatch finding is constructed.
- `node.text` is a `str`.
- `node.justifications` is a stable sequence.
- Stored hashes are either falsy “missing” values or comparable values such as strings.
- `supporting_justification` indices are valid whenever those nodes are used as antecedents.
- Missing antecedent references are silently ignored by content-hash recomputation.
- The graph is not mutated while `verify_all` is executing.
- Cyclic graphs are tolerated by the lower-level Merkle code via a sentinel empty-string hash rather than being reported as errors.

## Topics to Explore

- [function] `reasons/merkle.py:verify_justification` — Shows how individual justification hashes are recomputed and compared.
- [function] `reasons/merkle.py:compute_content_hash` — Explains how node text and antecedent Merkle hashes combine into a justification root.
- [file] `tests/test_merkle.py` — Provides expected behavior for clean, mutated, missing, and chained-hash cases.
- [general] `merkle-cycle-handling` — The lower-level hash code uses a sentinel for cycles, which affects verification results.
- [function] `reasons/merkle.py:backfill_hashes` — Complements `verify_all` by populating missing hashes before re-verification.

## Beliefs

- `verify-all-returns-summary-dict` — `verify_all` returns a dict with `findings` and `missing_hashes`, not a list as its docstring says.
- `verify-all-counts-falsy-stored-hashes` — `missing_hashes` counts every falsy node `text_hash` and every falsy justification `content_hash` encountered.
- `verify-all-does-not-mutate-net` — The function only mutates local variables and never writes computed hashes back into `net`.
- `verify-all-uses-shared-local-cache` — A single local `cache` is passed through all justification checks to reuse recomputed Merkle hashes.
- `verify-all-requires-sortable-node-ids` — The call to `sorted(net.nodes.items())` requires node IDs to be mutually orderable.
