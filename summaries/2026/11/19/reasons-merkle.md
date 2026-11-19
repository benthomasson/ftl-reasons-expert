---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/merkle.py

# `reasons/merkle.py`

## Purpose

This module owns **content-addressed integrity checks for the justification graph**. It computes SHA-256 hashes for node text and recursive Merkle-style hashes for justifications, verifies stored hashes against recomputed values, and backfills missing hashes.

It is a pure utility layer: no persistence, no CLI, no network I/O. It relies on the calling code to provide objects with the expected shape:

- `net.nodes` — mapping of node id to node objects
- node attributes: `.id`, `.text`, `.text_hash`, `.justifications`, `.supporting_justification`
- justification attributes: `.antecedents`, `.content_hash`

The module is imported by `reasons/api.py`, `reasons/network.py`, and `tests/test_merkle.py`.

---

## Key Components

### `compute_text_hash(text: str) -> str`

Returns the SHA-256 hex digest of UTF-8 encoded node text.

```python
sha256(text.encode("utf-8")).hexdigest()
```

This is the base hash for a node’s text.

---

### `_fresh_merkle_hash(node_id, net, cache, visiting)`

Private recursive worker for computing a node’s Merkle hash **from current text**, never from stored hashes.

Contract:

- If `node_id` is already in `cache`, return cached value.
- If `node_id` is already in `visiting`, treat it as a cycle and return `compute_text_hash("")`.
- Otherwise:
  - Read `node = net.nodes[node_id]`
  - Compute `text_hash = compute_text_hash(node.text)`
  - If the node has no justifications, or `node.supporting_justification is None`, the Merkle hash is just `text_hash`.
  - Otherwise, compute the content hash of the supporting justification and return:

```python
sha256((text_hash + "|" + supporting_content_hash).encode("utf-8")).hexdigest()
```

The result is stored in `cache[node_id]`.

Important: the node Merkle hash follows only `supporting_justification`. Other justifications do not affect a node’s Merkle hash.

---

### `_fresh_content_hash(text_hash, justification, net, cache, visiting)`

Private worker for computing a justification’s content hash.

It gathers Merkle hashes of all antecedents that exist in `net.nodes`:

```python
ant_hashes = sorted(
    _fresh_merkle_hash(a, net, cache, visiting)
    for a in justification.antecedents
    if a in net.nodes
)
```

Then:

- If there are antecedent hashes:

```python
parts = text_hash + "|" + "|".join(ant_hashes)
```

- If there are no antecedent hashes:

```python
parts = text_hash
```

Finally:

```python
sha256(parts.encode("utf-8")).hexdigest()
```

Notes:

- Antecedent order does not matter because hashes are sorted.
- Missing antecedents are silently ignored.
- Duplicate antecedents are not deduplicated.
- The incoming `text_hash` is expected to be computed from current node text.

---

### `compute_merkle_hash(node_id, net, cache=None)`

Public wrapper around `_fresh_merkle_hash`. Creates a fresh cache if needed and starts with an empty `visiting` set.

---

### `compute_content_hash(node, justification, net, cache=None)`

Public wrapper around `_fresh_content_hash`.

It recomputes `text_hash` from `node.text`:

```python
text_hash = compute_text_hash(node.text)
```

Note: the docstring says `sha256(node.text_hash | sorted(...))`, but the implementation uses the **current text hash**, not the stored `node.text_hash`. This is intentional for fresh verification, but the docstring is slightly misleading.

---

### `verify_node(node)`

Checks `node.text_hash` against `sha256(node.text)`.

Returns:

- `None` if `node.text_hash` is missing, or if the hash matches.
- A finding dict on mismatch:

```python
{
    "type": "text_mutation",
    "node_id": node.id,
    "stored_hash": node.text_hash,
    "current_hash": current,
}
```

---

### `verify_justification(node, j_index, net, cache=None)`

Checks `justification.content_hash` against the recomputed content hash.

Returns:

- `None` if `j_index` is out of range.
- `None` if the justification has no stored `content_hash`.
- A finding dict on mismatch:

```python
{
    "type": "chain_mutation",
    "node_id": node.id,
    "justification_index": j_index,
    "stored_hash": j.content_hash,
    "current_hash": current,
}
```

Out-of-range indices are silently ignored rather than raising.

---

### `verify_all(net)`

Iterates all nodes and all justifications, returning:

```python
{
    "findings": [...],
    "missing_hashes": int,
}
```

Behavior:

- Nodes are iterated in sorted order by node id.
- A shared `cache` is used across justifications to avoid recomputing antecedent Merkle hashes.
- Missing `text_hash` or `content_hash` values are counted in `missing_hashes` and are **not** added to `findings`.
- Justifications are still verified even if the node’s text hash is missing or mismatched.

---

### `backfill_hashes(net)`

Mutates the network by filling in only missing hashes:

- For each node with no `text_hash`, set `node.text_hash = compute_text_hash(node.text)`.
- For each justification with no `content_hash`, set it via `compute_content_hash(...)`.

Returns counts:

```python
{
    "nodes_updated": int,
    "justifications_updated": int,
}
```

It does **not** repair stale/mismatched hashes. It only fills missing ones.

---

## Patterns

- **Merkle DAG integrity**: Each node’s Merkle hash incorporates its text and the recursive Merkle hashes of its supporting justification’s antecedents.
- **Fresh recomputation**: Functions prefixed `_fresh_` deliberately ignore stored hashes and recompute from current text.
- **Memoization**: `cache` maps `node_id` to Merkle hash to avoid repeated recursive traversal.
- **Cycle guard**: `visiting` prevents infinite recursion on cyclic antecedent graphs. Back-edges return `sha256("")`.
- **Result dicts instead of exceptions**: Verification mismatches are returned as structured findings, not raised.
- **Separation of concerns**:
  - `compute_*` — calculate hashes
  - `verify_*` — compare stored vs current
  - `backfill_hashes` — mutate missing hash fields

---

## Dependencies

### Imports

Only standard library:

```python
import hashlib
```

### Implicit object protocol

The module depends on the caller’s network model providing:

- `net.nodes: Mapping[str, Node]`
- `Node.id`, `Node.text`, `Node.text_hash`
- `Node.justifications`
- `Node.supporting_justification`
- `Justification.antecedents`
- `Justification.content_hash`

### Imported by

- `reasons/api.py`
- `reasons/network.py`
- `tests/test_merkle.py`

This suggests it is exposed through the API and used during network integrity/sync workflows.

---

## Flow

### Computing a node Merkle hash

`compute_merkle_hash(node_id, net)`  
→ `_fresh_merkle_hash(node_id, net, cache, set())`  
→ checks cache  
→ checks cycle set  
→ computes current text hash  
→ if no supporting justification: returns text hash  
→ otherwise computes supporting justification content hash  
→ combines `text_hash + "|" + content_hash`  
→ hashes and caches result

### Computing a justification content hash

`compute_content_hash(node, justification, net)`  
→ recomputes `node.text` hash  
→ `_fresh_content_hash(...)`  
→ recursively computes Merkle hashes for antecedents present in `net.nodes`  
→ sorts antecedent hashes  
→ joins with `"|"`  
→ SHA-256 hashes the combined string

### Verification

`verify_all(net)`  
→ creates shared cache  
→ iterates `sorted(net.nodes.items())`  
→ verifies each node text hash  
→ counts missing text hashes  
→ verifies each justification content hash  
→ counts missing content hashes  
→ returns findings and missing count

### Backfill

`backfill_hashes(net)`  
→ iterates nodes  
→ fills missing `text_hash`  
→ fills missing `content_hash` for every justification  
→ returns update counts

---

## Invariants

- A stored `text_hash` is expected to equal `sha256(node.text)`.
- A stored `content_hash` is expected to equal the Merkle root over the node’s current text hash and sorted antecedent Merkle hashes.
- `_fresh_merkle_hash` and `_fresh_content_hash` never trust stored `text_hash` or `content_hash`.
- A node’s Merkle hash only follows `supporting_justification`; alternate justifications are independently verified but do not affect the node Merkle hash.
- Antecedent ordering does not affect content hashes because antecedent hashes are sorted.
- Antecedents not present in `net.nodes` are silently omitted from the content hash.
- Cycles are silently broken with `compute_text_hash("")`.
- Missing hashes are treated as “unverified”, not as mismatches.
- `backfill_hashes` only fills missing hashes; it never overwrites existing hashes.
- `verify_justification` silently ignores invalid `j_index` values.
- `_fresh_merkle_hash` assumes `supporting_justification` is either `None` or a valid index into `node.justifications`; otherwise it can raise `IndexError` or use Python negative indexing unexpectedly.
- The `cache` is only valid while the network’s text and justification structure remain unchanged.

---

## Error Handling

There is no `try/except` in this module.

- Hash mismatches are returned as finding dicts, not raised.
- Missing hashes are returned as counts / `None`, not raised.
- Invalid `j_index` in `verify_justification` returns `None`.
- Missing antecedents are silently filtered out.
- Cycles are silently handled by returning the hash of the empty string.
- Malformed network objects can raise:
  - `KeyError` if `net.nodes[node_id]` is missing
  - `AttributeError` if required attributes are absent
  - `IndexError` if `supporting_justification` is invalid
  - `TypeError` if `supporting_justification` is not an index/`None`
  - `UnicodeEncodeError` for non-UTF-8-encodable text
- `backfill_hashes` mutates objects in place and does not validate existing hashes.

---

## Topics to Explore

- [file] `reasons/network.py` — See how Merkle verification and backfilling are integrated into network synchronization or repair workflows.
- [file] `reasons/api.py` — Understand how this module’s verification/backfill functions are exposed to callers.
- [file] `tests/test_merkle.py` — Confirm expected hash formulas, mutation detection, missing-hash behavior, and cycle handling.
- [function] `reasons/merkle.py:verify_all` — Inspect how findings and `missing_hashes` are consumed by higher-level verification flows.
- [general] `merkle-cycle-handling` — Evaluate whether silently converting cycles to `sha256("")` is desired, or whether cycles should be reported as structural findings.

---

## Beliefs

- `fresh-hash-computation-ignores-stored-hashes` — `_fresh_merkle_hash` and `_fresh_content_hash` recompute hashes from current text and never read stored `text_hash` or `content_hash`.
- `merkle-hash-follows-supporting-justification-only` — A node’s Merkle hash only incorporates its `supporting_justification`; alternate justifications do not affect that node hash.
- `verify-all-counts-missing-hashes-separately-from-findings` — `verify_all` reports missing text/content hashes in `missing_hashes` and does not include them in `findings`.
- `backfill-hashes-only-fills-missing-hashes` — `backfill_hashes` computes only missing `text_hash`/`content_hash` values and leaves existing hashes untouched.
- `cycle-guard-returns-empty-string-hash` — A back-edge encountered during recursive Merkle computation returns `compute_text_hash("")` rather than raising or reporting a cycle.
