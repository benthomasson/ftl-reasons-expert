---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Function: compute_merkle_hash in reasons/merkle.py

# `compute_merkle_hash` Explanation

## Purpose

`compute_merkle_hash(node_id, net, cache=None)` is the public entry point for computing a node’s recursive Merkle hash from the **current in-memory text and justification graph**, not from stored hashes.

It is a thin wrapper around `_fresh_merkle_hash`. Its job is to:

1. Create a fresh memoization cache if one is not supplied.
2. Create a fresh cycle-detection `visiting` set.
3. Delegate the actual recursive computation.

This exists so callers can compute a node-level Merkle hash that incorporates the node’s text and, if it has a supporting justification, the recursive hashes of that justification’s antecedents. Any text mutation anywhere in the supporting chain should change the resulting hash.

## Contract

### Preconditions

- `node_id` should be a key in `net.nodes`.
- `net.nodes[node_id]` must expose at least:
  - `.text`
  - `.justifications`
  - `.supporting_justification`
- If `supporting_justification` is not `None` and `node.justifications` is non-empty, it should be a valid index into `node.justifications`.
- Each selected justification must expose `.antecedents`, an iterable of node IDs.
- Antecedent IDs may be missing from `net.nodes`; those are silently filtered out during content hashing.
- If `cache` is supplied, it must be a mutable mapping from node ID to hash string. Existing entries are trusted as-is.

### Postconditions

- Returns a SHA-256 hex digest string for `node_id`.
- If `cache` was supplied, it is mutated with computed hashes for nodes visited during the call.
- Does not mutate `net`, nodes, justifications, or stored hash fields such as `node.text_hash` or `justification.content_hash`.
- Uses current `.text` values, not stored hashes.

### Invariants

- Cache lookup happens before cycle detection.
- Cycle re-entry returns `compute_text_hash("")`, the SHA-256 hash of the empty string.
- For nodes with a usable supporting justification:
  - The node hash wraps a content hash.
  - The content hash includes the node’s text hash and sorted antecedent Merkle hashes.
- For nodes without a usable supporting justification, the Merkle hash is just the node text hash.
- Missing antecedent IDs are omitted from the content hash.

## Parameters

### `node_id`

The ID of the node to hash. It must be usable as a key into `net.nodes`. In this codebase it is assumed to be hashable, typically a string.

If the ID is not present in `net.nodes`, a `KeyError` propagates from `_fresh_merkle_hash`.

### `net`

The network/graph object. The function assumes:

- `net.nodes` supports membership checks and key lookup.
- Each node has `.text`, `.justifications`, and `.supporting_justification`.
- Each justification has `.antecedents`.

The function does not validate this shape.

### `cache=None`

Optional memoization dictionary mapping node IDs to already-computed Merkle hashes.

Behavior:

- If `None`, a new empty dictionary is created and discarded after the call.
- If provided, it is used and mutated.
- Existing cache entries are trusted without checking whether they match the current graph state.

This last point is important: passing a stale cache after mutating text can produce stale hashes.

## Return Value

Returns a `str` containing the SHA-256 hex digest of the computed Merkle hash.

For a leaf node with no supporting justification:

```text
M(node) = SHA256(node.text)
```

For a node with a supporting justification:

```text
text_hash = SHA256(node.text)

content_hash = SHA256(
    text_hash + "|" + "|".join(sorted(antecedent Merkle hashes))
)
# or SHA256(text_hash) if there are no antecedents

M(node) = SHA256(text_hash + "|" + content_hash)
```

The caller must handle exceptions for malformed graphs. The function does not return an error dictionary.

## Algorithm

`compute_merkle_hash` itself does very little:

1. If `cache is None`, create `{}`.
2. Call `_fresh_merkle_hash(node_id, net, cache, set())`.
3. Return its result.

The real recursion happens in `_fresh_merkle_hash`:

1. If `node_id` is already in `cache`, return the cached hash immediately.
2. If `node_id` is already in `visiting`, return `compute_text_hash("")`.
   - This prevents infinite recursion on cycles.
   - The result is not cached under the revisited node.
3. Add `node_id` to `visiting`.
4. Fetch `node = net.nodes[node_id]`.
5. Compute `text_hash = compute_text_hash(node.text)`.
6. If the node has no justifications, or `supporting_justification is None`:
   - Result is `text_hash`.
7. Otherwise:
   - Select `j = node.justifications[node.supporting_justification]`.
   - Compute `ch = _fresh_content_hash(text_hash, j, net, cache, visiting)`.
   - Result is `SHA256(text_hash + "|" + ch)`.
8. Store result in `cache[node_id]`.
9. Return result.

`_fresh_content_hash` works like this:

1. For each antecedent ID in `justification.antecedents` that exists in `net.nodes`, recursively compute that antecedent’s Merkle hash using the same `cache` and `visiting` set.
2. Sort the resulting antecedent hashes.
3. Build:
   - `parts = text_hash + "|" + "|".join(ant_hashes)` if there are antecedents.
   - `parts = text_hash` if there are no antecedents.
4. Return `SHA256(parts)`.

Two subtleties are worth calling out:

- Antecedent order does not matter because hashes are sorted.
- Duplicate antecedent IDs are not deduplicated; duplicates produce duplicate hashes in the joined list.
- Missing antecedent IDs are silently ignored.

## Side Effects

- Mutates the supplied `cache` dictionary by adding computed node hashes.
- Creates and mutates a local `visiting` set during recursion.
- Makes no network, filesystem, or I/O calls.
- Does not update stored hashes on nodes or justifications.
- Not thread-safe if the same `cache` is shared concurrently.

## Error Handling

There is no explicit error handling in `compute_merkle_hash`.

Exceptions propagate from the helpers:

- `KeyError` if `node_id` or a traversed node ID is missing from `net.nodes`.
- `IndexError` if `supporting_justification` is an out-of-range positive index.
- `TypeError` if `supporting_justification` is not a valid list index type.
- `AttributeError` if the node or justification objects lack expected attributes.
- `RecursionError` for very deep acyclic chains.
- `TypeError` if `cache` is not a mutable mapping as expected.

Cycles do not raise. A node already in `visiting` is replaced with the empty-string text hash. Missing antecedent IDs do not raise inside `_fresh_content_hash`; they are filtered out.

## Usage Patterns

Typical use:

```python
hash_value = compute_merkle_hash(node_id, net)
```

For batch computation:

```python
cache = {}
h1 = compute_merkle_hash(nid1, net, cache)
h2 = compute_merkle_hash(nid2, net, cache)
```

Caller obligations:

- Do not reuse a cache across graph mutations or different `net` instances unless you know the cached entries are still valid.
- Treat the result as derived from the current graph state, not as verification of stored hashes.
- Catch exceptions if the graph may be malformed.
- Use `verify_justification`, `verify_all`, or `compute_content_hash` when you need justification-level content-hash verification rather than a node-level Merkle hash.

## Dependencies

Internal dependencies:

- `compute_text_hash`
- `_fresh_merkle_hash`
- `_fresh_content_hash`

External dependency:

- `hashlib`

Data-model dependencies:

- `net.nodes`
- `node.text`
- `node.justifications`
- `node.supporting_justification`
- `justification.antecedents`

## Topics to Explore

- [file] `reasons/merkle.py` — Full hashing and verification flow, including `compute_content_hash`, `verify_justification`, and `backfill_hashes`.
- [function] `reasons/merkle.py:_fresh_merkle_hash` — Core recursive node-hash algorithm, including cycle handling and cache behavior.
- [function] `reasons/merkle.py:_fresh_content_hash` — How antecedent hashes are recursively gathered, sorted, and combined.
- [file] `tests/test_merkle.py` — Test coverage for cycles, missing antecedents, text mutation, and hash backfilling.
- [general] `merkle-cache-semantics` — Why reusing a cached hash after graph mutation can produce stale results, and how cache invalidation should be handled.

## Beliefs

- `compute-merkle-hash-ignores-stored-hashes` — `compute_merkle_hash` recomputes from current text and does not read `node.text_hash` or `justification.content_hash`.
- `cache-parameter-is-mutated-and-trusted` — A non-`None` cache is used for memoization and existing entries are returned without validation against current graph state.
- `node-hash-uses-only-supporting-justification` — For nodes with a non-`None` supporting justification, only that justification’s antecedents contribute to the node’s Merkle hash.
- `missing-antecedents-are-silently-skipped` — `_fresh_content_hash` filters antecedent IDs not present in `net.nodes`, so dangling references do not raise there.
- `cycles-resolve-to-empty-string-hash` — When recursion revisits a node already in the `visiting` set, it substitutes `compute_text_hash("")` for that node’s hash.
