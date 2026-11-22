---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: compute_content_hash in reasons/merkle.py

`compute_content_hash` is the public entry point for recomputing a justification’s content hash from the current in-memory text, rather than trusting stored hashes.

## Purpose

It computes the Merkle-style hash for one justification attached to a node:

```python
sha256(node_text_hash | sorted(antecedent_merkle_hashes))
```

Important detail: despite the docstring saying `node.text_hash`, the implementation does **not** read `node.text_hash`. It recomputes the text hash from `node.text` using `compute_text_hash(node.text)`. This is what makes it useful for detecting text mutation during verification.

The function exists so callers can:

- recompute a justification’s content hash and compare it to `justification.content_hash`
- backfill missing `content_hash` values
- detect chain mutation when any text or antecedent Merkle hash changes

## Contract

Preconditions:

- `node` must expose `.text`, expected to be a `str`.
- `justification` must expose `.antecedents`, expected to be an iterable of node IDs.
- `net` must expose `.nodes`, a mapping from node ID to node objects.
- Any antecedent node reached recursively must expose `.text`, `.justifications`, and `.supporting_justification`.
- If a reached node has non-empty `.justifications` and `.supporting_justification` is not `None`, then `.supporting_justification` must be a valid index into that node’s `.justifications` list.
- `cache`, if provided, must be a mutable mapping from node ID to Merkle hash string. Cached values are trusted and reused without recomputation.
- The antecedent graph is assumed to be acyclic for canonical behavior. Cycles are not rejected; they are broken with a placeholder hash.
- The caller is expected to pass the correct `node` / `justification` pair. This function does not verify that `justification` actually belongs to `node.justifications`.

Postconditions:

- Returns a lowercase 64-character SHA-256 hex digest.
- Does not mutate `node.text_hash` or `justification.content_hash`.
- May mutate `cache` by inserting Merkle hashes for antecedent nodes visited during recursion.
- Missing antecedent IDs are silently skipped because `_fresh_content_hash` filters with `if a in net.nodes`.
- The current node’s text is always rehashed from `node.text`; cached/stored text hashes are not used.

Invariants:

- Antecedent Merkle hashes are sorted lexicographically before being combined.
- The final content hash is computed from ASCII hex-string material, not raw bytes.
- If there are no antecedent hashes, the input to the final SHA-256 is just `text_hash`.
- If there are antecedent hashes, the input is `text_hash + "|" + "|".join(sorted_hashes)`.
- Cached antecedent hashes are trusted; a stale cache produces stale antecedent components.

## Parameters

- `node`: object representing the node whose text is being hashed. Only `.text` is used by this function. The node’s stored `.text_hash` is ignored.
- `justification`: object representing the justification being hashed. Its `.antecedents` collection determines the immediate child node IDs. Antecedents absent from `net.nodes` are ignored. Duplicate antecedent IDs are not deduplicated, so duplicate hashes will appear multiple times in the sorted hash list.
- `net`: object with `.nodes`, mapping node IDs to node objects. This is used recursively by `_fresh_merkle_hash` to resolve antecedent nodes.
- `cache`: optional mutable mapping. If `None`, a new empty dict is created. If supplied, it is mutated and shared across recursive calls. The cache stores node-level Merkle hashes keyed by node ID, not justification content hashes.

Edge cases:

- Empty `justification.antecedents`: content hash is `sha256(text_hash).hexdigest()`.
- Missing antecedent IDs: skipped.
- Duplicate antecedent IDs: repeated in the hash material.
- Cyclic antecedent references: cycle detection returns `compute_text_hash("")` as a placeholder for the already-visited node.
- A real node with empty text has the same text hash as the cycle placeholder.
- Invalid `supporting_justification` index can raise `IndexError`.
- Negative `supporting_justification` is accepted by Python list indexing and is not validated.
- Stale or cross-network `cache` produces incorrect results.

## Return Value

Returns a SHA-256 hex digest string representing the justification’s content hash.

It never returns `None`. It does not return an error object. If inputs are malformed or the graph contains invalid indices/attributes, exceptions propagate to the caller.

Callers typically compare the return value to `justification.content_hash`, as `verify_justification` does, or assign it to a missing `content_hash`, as `backfill_hashes` does.

## Algorithm

1. If `cache is None`, create a fresh empty dict.
2. Compute the current node text hash:
   ```python
   text_hash = compute_text_hash(node.text)
   ```
   This is a fresh SHA-256 of the UTF-8 encoded text.
3. Call `_fresh_content_hash(text_hash, justification, net, cache, set())`.
4. Inside `_fresh_content_hash`:
   - For each antecedent ID in `justification.antecedents` that exists in `net.nodes`, compute that antecedent node’s Merkle hash via `_fresh_merkle_hash`.
   - Sort the resulting antecedent hashes lexicographically.
   - If any antecedent hashes exist, build:
     ```python
     text_hash + "|" + "|".join(sorted_antecedent_hashes)
     ```
     Otherwise use just `text_hash`.
   - Return `sha256(parts.encode("utf-8")).hexdigest()`.
5. Inside `_fresh_merkle_hash`, for each antecedent node:
   - If the node ID is already in `cache`, return the cached Merkle hash immediately.
   - If the node ID is in `visiting` and not cached, return `compute_text_hash("")` as a cycle placeholder.
   - Add the node ID to `visiting`.
   - Fetch the node from `net.nodes[node_id]`.
   - Compute the node’s current text hash.
   - If the node has no justifications or `supporting_justification is None`, its Merkle hash is just its text hash.
   - Otherwise, select:
     ```python
     j = node.justifications[node.supporting_justification]
     ```
     compute that supporting justification’s content hash via `_fresh_content_hash`, and return:
     ```python
     sha256(text_hash + "|" + content_hash).hexdigest()
     ```
   - Store the resulting node Merkle hash in `cache`.

So the current justification’s immediate antecedents are hashed using each antecedent node’s own Merkle hash. That antecedent Merkle hash follows the antecedent node’s `supporting_justification`, recursively. The current node’s `supporting_justification` is not used directly here; it only matters when this node is reached as an antecedent of something else.

## Side Effects

- Mutates the supplied `cache` by adding node-level Merkle hashes for nodes visited during recursive antecedent resolution.
- Does not write to `node.text_hash`.
- Does not write to `justification.content_hash`.
- Does not perform I/O.
- Does not otherwise mutate `node`, `justification`, or `net`.
- Uses recursion, so very deep chains can raise `RecursionError`.

## Error Handling

There is no explicit error handling. Exceptions propagate.

Possible exceptions include:

- `AttributeError` if `node`, `net`, or reached antecedent nodes lack expected attributes.
- `TypeError` if `justification.antecedents` is not iterable, if text is not `str`-like, or if an antecedent ID is unhashable.
- `IndexError` if a reached node has non-empty `.justifications` and `.supporting_justification` is out of range.
- `KeyError` is normally avoided for missing immediate antecedents because `_fresh_content_hash` filters on `a in net.nodes`, but it could still occur if `net.nodes` changes during traversal or has inconsistent membership behavior.
- `RecursionError` for very deep acyclic graphs.
- `UnicodeEncodeError` if `node.text` contains invalid surrogate data that cannot be UTF-8 encoded.

Missing antecedent IDs are not treated as errors; they are silently skipped. Cycles are not treated as errors; they are converted into the empty-string text hash placeholder.

## Usage Patterns

Typical callers:

- `verify_justification(node, j_index, net, cache)` recomputes the current content hash and compares it with `j.content_hash`.
- `backfill_hashes(net)` computes and stores missing `j.content_hash` values.
- `verify_all(net)` shares one cache across the whole verification pass.

Caller obligations:

- Ensure the cache is fresh for the current network state. If node text or justification structure changes, invalidate the cache.
- Do not reuse a cache across different `net` objects with overlapping node IDs.
- Ensure `supporting_justification` indexes are valid for any node reachable as an antecedent.
- Treat cyclic antecedent graphs as non-canonical unless the placeholder cycle behavior is explicitly acceptable.
- Use the returned value; this function does not persist it.

## Dependencies

- Standard library `hashlib`.
- `compute_text_hash` from the same module.
- Internal recursive helpers `_fresh_content_hash` and `_fresh_merkle_hash`.
- Data model assumptions: `node.text`, `node.justifications`, `node.supporting_justification`, `justification.antecedents`, and `net.nodes`.

## Topics to Explore

- [function] `reasons/merkle.py:_fresh_merkle_hash` — Shows how node-level Merkle hashes recurse through `supporting_justification` and how cycles are broken.
- [function] `reasons/merkle.py:verify_justification` — Demonstrates how `compute_content_hash` is used to compare live hashes against stored `content_hash` values.
- [file] `tests/test_merkle.py` — Likely covers expected hash determinism, mutation detection, missing hashes, and edge cases.
- [general] `merkle-cycle-handling` — The empty-string placeholder behavior and why cyclic antecedent graphs may become order-dependent.
- [general] `justification-graph-invariants` — Assumptions such as acyclicity, valid `supporting_justification` indexes, and missing antecedent handling.

## Beliefs

- `compute-content-hash-ignores-stored-text-hash` — It recomputes text hashes from `node.text` and never reads `node.text_hash`.
- `compute-content-hash-mutates-cache` — It populates the provided cache with node-level Merkle hashes, but not the final justification content hash.
- `missing-antecedents-are-skipped` — Antecedent IDs absent from `net.nodes` are silently omitted from the hash input.
- `cycle-placeholder-is-empty-string-hash` — A node encountered while already in `visiting` and not cached is replaced with `compute_text_hash("")`.
- `cache-values-are-trusted-without-recomputation` — Cached node Merkle hashes are returned directly, so stale cache entries produce stale antecedent hashes.
