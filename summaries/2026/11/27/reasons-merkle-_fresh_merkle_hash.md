---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Function: _fresh_merkle_hash in reasons/merkle.py

## `_fresh_merkle_hash`

### Purpose

`_fresh_merkle_hash` is the recursive core of the Merkle hashing logic. It computes a node’s Merkle hash using the node’s **current text** and the current text of its supporting justification’s antecedents. It deliberately ignores stored integrity fields such as `node.text_hash` and `justification.content_hash`.

It exists so verification and backfill can recompute what the hash *should be now*, rather than trusting stored hashes that may be stale or corrupted by a text mutation.

It is mutually recursive with `_fresh_content_hash`: a node hash may depend on a justification content hash, and a justification content hash depends on the Merkle hashes of its antecedents.

---

### Contract

**Preconditions**

- `node_id` must be hashable because it is used as a key in `cache` and as a member of `visiting`.
- Normally `node_id` must exist in `net.nodes`, unless the function returns early because:
  - `node_id in cache`, or
  - `node_id in visiting`.
- `net.nodes` must behave like a mapping from node IDs to node objects.
- Each node must expose:
  - `.text`
  - `.justifications`
  - `.supporting_justification`
- Each justification must expose:
  - `.antecedents`
- `node.text` must be a `str`, because `compute_text_hash` calls `.encode("utf-8")`.
- `node.supporting_justification` must be either `None` or a valid index into `node.justifications`. The code does not explicitly validate the index.
- `cache` must be a mutable mapping.
- `visiting` must be a mutable set.
- If `cache` is reused after the underlying `net` changes, the caller must invalidate or replace it; otherwise stale hashes will be returned.

**Postconditions**

- Returns a SHA-256 hex digest string for the node’s current Merkle hash.
- Stores the computed result in `cache[node_id]` unless returning early due to an existing cache entry or cycle detection.
- Adds `node_id` to `visiting` before doing deeper computation.
- Does **not** remove `node_id` from `visiting` after computation.
- Does not read or compare `node.text_hash` or `justification.content_hash`.

**Key invariant**

The cache is checked **before** cycle detection. Therefore, if a node is both cached and present in `visiting`, the cached value is returned. This is important because completed nodes remain in `visiting`.

---

### Parameters

| Parameter | Meaning |
|---|---|
| `node_id` | Identifier of the node to hash. It is used as a cache key and cycle-detection set member. |
| `net` | Object containing a `nodes` mapping. The function reads `net.nodes[node_id]` and node/justification fields from it. |
| `cache` | Mutable mapping from node ID to already-computed Merkle hash. Used to avoid recomputation within a traversal. |
| `visiting` | Mutable set of node IDs currently considered active in the recursive traversal. Used to break cycles. |

Edge cases:

- If `node_id` is missing from `net.nodes` and is neither cached nor visiting, `net.nodes[node_id]` raises `KeyError`.
- If `node_id` is already in `visiting`, the function returns `compute_text_hash("")` rather than raising or recursing forever.
- If `node_id` is already in `cache`, the function returns that value without consulting `net` or `visiting`.
- `visiting` is mutated and not cleaned up. It is intended to be a fresh set per top-level computation.
- `cache` can be shared across multiple nodes/justifications in higher-level operations such as `verify_all` and `backfill_hashes`.

---

### Return Value

Returns a lowercase SHA-256 hex digest string.

There are several cases:

1. **Cached node**
   - Returns `cache[node_id]` immediately.

2. **Cycle detected**
   - If `node_id in visiting`, returns:
     ```python
     compute_text_hash("")
     ```
   - This is the SHA-256 hash of the empty string, used as a deterministic placeholder to break recursion.
   - This placeholder is not cached for `node_id`.

3. **Node has no justifications or no supporting justification**
   - Returns `text_hash`, where:
     ```python
     text_hash = compute_text_hash(node.text)
     ```

4. **Node has a supporting justification**
   - Fetches:
     ```python
     j = node.justifications[node.supporting_justification]
     ```
   - Computes:
     ```python
     ch = _fresh_content_hash(text_hash, j, net, cache, visiting)
     ```
   - Returns:
     ```python
     hashlib.sha256(
         (text_hash + "|" + ch).encode("utf-8")
     ).hexdigest()
     ```

Note that this means a node with a supporting justification is hashed as:

```text
sha256(text_hash + "|" + content_hash)
```

where `content_hash` itself is computed by `_fresh_content_hash`. This is effectively a nested/double hash involving the node text hash.

---

### Algorithm

Step by step:

1. Check whether `node_id` is already in `cache`.
   - If yes, return `cache[node_id]`.

2. Check whether `node_id` is already in `visiting`.
   - If yes, return `compute_text_hash("")`.
   - This breaks recursive cycles.

3. Add `node_id` to `visiting`.

4. Fetch the node:
   ```python
   node = net.nodes[node_id]
   ```

5. Compute the node’s current text hash:
   ```python
   text_hash = compute_text_hash(node.text)
   ```

6. Determine whether the node participates in a supporting justification:
   ```python
   if not node.justifications or node.supporting_justification is None:
       result = text_hash
   ```
   In this case, the node’s Merkle hash is simply its text hash. Stored justifications are ignored if there is no supporting justification.

7. Otherwise, select the supporting justification:
   ```python
   j = node.justifications[node.supporting_justification]
   ```

8. Compute that justification’s content hash from current text:
   ```python
   ch = _fresh_content_hash(text_hash, j, net, cache, visiting)
   ```

9. Combine the node text hash and justification content hash:
   ```python
   result = hashlib.sha256(
       (text_hash + "|" + ch).encode("utf-8")
   ).hexdigest()
   ```

10. Store and return:
    ```python
    cache[node_id] = result
    return result
    ```

For context, `_fresh_content_hash` does roughly:

```python
ant_hashes = sorted(
    _fresh_merkle_hash(a, net, cache, visiting)
    for a in justification.antecedents
    if a in net.nodes
)
parts = text_hash + "|" + "|".join(ant_hashes) if ant_hashes else text_hash
return hashlib.sha256(parts.encode("utf-8")).hexdigest()
```

So missing antecedents are silently skipped, and antecedent order is normalized by sorting the resulting hashes.

---

### Side Effects

- Mutates `cache` by adding computed node hashes.
- Mutates `visiting` by adding `node_id`; it never removes entries.
- Reads from `net.nodes`, `node.text`, `node.justifications`, and `justification.antecedents`.
- Recursively mutates the same `cache` and `visiting` objects through calls to `_fresh_content_hash`.
- Performs no I/O, logging, network access, or external calls.
- Is not thread-safe if `cache` or `visiting` are shared across threads.

---

### Error Handling

There is no `try`/`except`. Exceptions propagate to the caller.

Likely exceptions:

- `KeyError` if `net.nodes[node_id]` is accessed for a missing node.
- `IndexError` if `node.supporting_justification` is an invalid positive index.
- Negative indices are accepted by Python and may select from the end of `node.justifications`, which is probably unintended unless explicitly allowed.
- `TypeError` if `supporting_justification` is not a valid index type, or if node IDs/antecedents are unhashable.
- `AttributeError` if `node.text` is not a string, or if expected fields are missing.
- `RecursionError` for very deep acyclic justification chains.
- If an exception occurs after adding `node_id` to `visiting`, the set is not cleaned up.
- Cyclic references are handled by returning the empty-string hash rather than raising an exception.
- `_fresh_content_hash` silently ignores antecedents that are not present in `net.nodes`.

---

### Usage Patterns

This is an internal helper. The public entry point is typically:

```python
compute_merkle_hash(node_id, net, cache=None)
```

which creates a fresh cache and fresh `visiting` set by default:

```python
def compute_merkle_hash(node_id, net, cache=None):
    if cache is None:
        cache = {}
    return _fresh_merkle_hash(node_id, net, cache, set())
```

It is also called indirectly through `_fresh_content_hash` when computing justification content hashes and when verifying or backfilling hashes.

Caller obligations:

- Do not reuse a `cache` after mutating node text or justification structure unless you intend to use stale results.
- Do not assume `visiting` is cleaned after the call.
- Do not rely on cycle hashes being canonical across different traversal roots. Because cycle detection returns `compute_text_hash("")` and because `visiting` is not pruned, cyclic graphs can produce traversal-order-dependent hashes.
- Missing antecedents are not errors; they are omitted from the content hash. Callers must decide whether dangling antecedent IDs should be validated separately.

---

### Dependencies

Internal:

- `compute_text_hash`
- `_fresh_content_hash`
- Node model fields:
  - `node.text`
  - `node.justifications`
  - `node.supporting_justification`
- Justification model field:
  - `justification.antecedents`
- `net.nodes`

External:

- `hashlib` from the Python standard library.

---

### Assumptions Not Enforced by the Type System

- `node_id` exists in `net.nodes` unless it is cached or visiting.
- `supporting_justification` is either `None` or a valid index into `node.justifications`.
- Negative `supporting_justification` values are technically accepted by Python but likely not intended.
- `node.text` is a `str`.
- `node.justifications` is indexable and supports integer indexing.
- `justification.antecedents` is iterable and contains hashable node IDs.
- Missing antecedent IDs are tolerated and silently filtered out.
- `cache` and `visiting` are fresh and consistent with the current `net`; the function does not validate or clear them.
- Cyclic justification graphs are handled structurally, but not necessarily with a canonical cycle hash.
- The recursion depth is sufficient for the justification graph depth.

---

## Topics to Explore

- [file] `reasons/merkle.py` — Full hashing, verification, and backfill implementation for justification chains.
- [function] `reasons/merkle.py:_fresh_content_hash` — Shows how antecedent hashes are sorted, joined, and how missing antecedents are skipped.
- [function] `reasons/merkle.py:verify_all` — Demonstrates shared-cache traversal and how mutation findings are collected.
- [file] `tests/test_merkle.py` — Tests likely cover expected hashes, mutation detection, and cycle behavior.
- [general] `cycle-handling-in-merkle-hash` — Worth exploring because active cycles return `compute_text_hash("")`, and `visiting` is never pruned.

## Beliefs

- `fresh-merkle-hash-ignores-stored-hashes` — `_fresh_merkle_hash` computes from `node.text` and recursive current-state hashes; it never reads `node.text_hash` or `justification.content_hash`.
- `cycle-detection-returns-empty-string-hash` — When recursion re-enters a node already in `visiting`, the function returns `compute_text_hash("")` instead of raising or recursing further.
- `visiting-set-is-not-pruned` — `_fresh_merkle_hash` adds each computed node to `visiting` and never removes it after caching.
- `cache-takes-precedence-over-visiting` — The cache lookup occurs before the cycle check, so a cached node returns its cached hash even if it is also in `visiting`.
- `missing-antecedents-are-skipped` — `_fresh_content_hash` filters antecedents to those present in `net.nodes`, silently omitting dangling IDs from the Merkle root.
