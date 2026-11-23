---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Function: hash_sources in reasons/check_stale.py

## `hash_sources`

### Purpose

`hash_sources` backfills `node.source_hash` for nodes that have a source path but do not yet have a stored content hash. It can also force re-hashing of all nodes with sources. It is not a staleness checker itself: it does not compare old vs. new hashes or report stale nodes. It simply reads each resolvable source file, computes its full SHA-256 hash, and writes that hash onto the node in memory.

This is useful before running staleness checks: nodes with no `source_hash` cannot be compared, so `hash_sources` populates them. `force=True` is for cases where the caller has already decided that current source content should become the new baseline.

---

### Contract

**Preconditions**

- `network.nodes` exists and maps node IDs to node-like objects.
- Each visited node must expose:
  - `source`
  - `source_hash`
  - `metadata`
- If `metadata` is truthy, it must support `.get(...)`, normally a `dict`.
- `network.repos`, if used, must map names to path-like values.
- Any source path that resolves must be readable as a regular file; the function does not validate that it is not a directory.

**Postconditions**

- For every node included in the returned list, `node.source_hash` has been assigned the newly computed full SHA-256 hash.
- The returned list contains one dict per node that was actually hashed, in sorted node-ID order.
- Nodes with no `source`, already-hashed nodes when `force=False`, or sources that cannot be resolved are skipped and are not included in the result list.
- The function does **not** save the network or persist changes; it only mutates the in-memory `Network`.
- If an exception occurs partway through, earlier nodes may already have been mutated.

**Important invariants / assumptions**

- The function does **not** filter by `node.truth_value`. Unlike `check_stale` and `pin_sources`, it processes all nodes with a truthy `source`, including nodes that are not `IN`. This is not enforced by the type system.
- `node.source_hash` is treated as a string-like value whose truthiness means “already has a hash.” An empty string, `None`, or other falsy value is treated as “missing.”
- `was_empty` is computed from the old value before mutation, not from whether the file content changed.
- `force=True` does not compare old and new hashes; it simply overwrites and records the node again.

---

### Parameters

- `network: Network`
  - The graph/network containing nodes.
  - Used for `network.nodes` and optionally `network.repos`.
  - Mutated in place through each node’s `source_hash`.

- `repos: dict[str, Path] | None = None`
  - Optional mapping from repo name to repository root path.
  - If omitted and `network.repos` is truthy, a local copy is built:
    ```python
    repos = {k: Path(v) for k, v in network.repos.items()}
    ```
  - If `repos` is explicitly passed, it is used as-is; `network.repos` is not merged in.
  - An explicitly passed empty dict is not replaced by `network.repos`.

- `force: bool = False`
  - If `False`, skip any node whose `source_hash` is truthy.
  - If `True`, re-hash every node with a truthy `source` and resolvable path, even if it already has a hash.
  - `was_empty` still records whether the previous hash was missing.

- `db_dir: Path | None = None`
  - Passed through to `resolve_source_path`.
  - Used as a location to resolve sources that live next to the reasons database.

---

### Return Value

Returns `list[dict]`.

Each dict has exactly these keys:

```python
{
    "node_id": str,
    "source": str,
    "hash": str,
    "was_empty": bool,
}
```

- `node_id`: the node ID from `network.nodes`.
- `source`: the original `node.source` string.
- `hash`: the full SHA-256 hex digest written to `node.source_hash`.
- `was_empty`: `True` if `node.source_hash` was falsy before assignment; `False` if an existing hash was overwritten under `force=True`.

The list is ordered by sorted `network.nodes.items()` iteration order.

The caller must handle:
- Persisting the mutated `Network`.
- Deciding whether processing non-`IN` nodes is acceptable.
- Handling the fact that skipped nodes produce no result entries.
- Handling partial mutation if an exception occurs mid-loop.

---

### Algorithm

1. If `repos is None` and `network.repos` is truthy, convert `network.repos` values to `Path` objects into a local `repos` mapping.

2. Initialize `results = []`.

3. Iterate over `sorted(network.nodes.items())`.

4. For each `(nid, node)`:
   - Skip if `node.source` is falsy.
   - Skip if `node.source_hash` is truthy and `force` is `False`.
   - Extract `agent` from metadata:
     ```python
     agent = node.metadata.get("agent") if node.metadata else None
     ```
   - Resolve the source path:
     ```python
     path = resolve_source_path(node.source, repos, db_dir, agent=agent)
     ```
   - Skip if `path is None`.
   - Compute the full SHA-256 hash:
     ```python
     new_hash = hash_file(path)
     ```
   - Record whether the old hash was empty:
     ```python
     was_empty = not node.source_hash
     ```
   - Mutate the node:
     ```python
     node.source_hash = new_hash
     ```
   - Append the result dict.

5. Return `results`.

---

### Side Effects

- Reads source files from disk via `hash_file(path)`, which calls:
  ```python
  hashlib.sha256(path.read_bytes()).hexdigest()
  ```
- Mutates `node.source_hash` on every successfully resolved and hashed node.
- Does **not** mutate `network.repos`, `node.metadata`, `node.source`, or any other node field.
- Does **not** save or serialize the network.
- Does **not** call git, the database, or any external process.
- Because it mutates during iteration, an exception on a later node does not roll back earlier mutations.

---

### Error Handling

There is no explicit `try/except` in `hash_sources`.

Possible outcomes:

- Missing or unresolved source path:
  - `resolve_source_path` returns `None`.
  - The node is silently skipped.
- Existing hash with `force=False`:
  - The node is silently skipped.
- Unreadable file, permission error, path is a directory, or race condition after `exists()` check:
  - `path.read_bytes()` raises an `OSError` subclass such as `PermissionError`, `IsADirectoryError`, or `FileNotFoundError`.
  - This propagates out of `hash_sources`.
- Malformed `node.metadata` that is truthy but lacks `.get`:
  - `AttributeError` propagates.
- Node IDs that are not mutually sortable:
  - `sorted(network.nodes.items())` can raise `TypeError`.
- Any partially completed loop may already have updated earlier nodes.

So error handling is mostly “skip if resolvable path is absent; otherwise let filesystem/runtime errors propagate.”

---

### Usage Patterns

Typical usage likely looks like:

```python
results = hash_sources(network, repos=repos, db_dir=db_dir)
# caller must save network
network.save(...)
```

Common patterns:

- Backfill missing hashes before a staleness audit.
- Force-rehash after intentionally accepting changed source files:
  ```python
  hash_sources(network, force=True)
  ```
- Inspect the returned list to report how many nodes were newly hashed vs. re-hashed:
  - `was_empty=True` means the node had no prior hash.
  - `was_empty=False` with `force=True` means an existing hash was overwritten.

Caller obligations:

- Persist the mutated network if the new hashes should survive.
- Be aware that non-`IN` nodes may be processed.
- Avoid calling with `force=True` unless the current file content is intended to become the new baseline.
- Ensure source paths point to files, not directories, or handle filesystem exceptions.

---

### Dependencies

- `hashlib` for SHA-256.
- `Path` from `pathlib` for filesystem access.
- `Network` from `.network` for the type annotation and runtime object shape.
- `resolve_source_path` from the same module for source resolution.
- `hash_file` from the same module for full-file hashing.

`hash_sources` relies indirectly on the `Network` and node model, especially:
- `network.nodes`
- `node.source`
- `node.source_hash`
- `node.metadata`
- `network.repos`

---

### Assumptions Not Enforced by the Type System

- `node.truth_value` is not checked; all nodes with sources can be hashed.
- `node.metadata` is assumed to be dict-like if truthy.
- `node.source_hash` is assumed to be a mutable attribute on the node.
- A resolved path is assumed to be a regular file.
- `network.nodes` keys are assumed to be sortable.
- Callers are assumed to persist the network after mutation.
- `force=True` is assumed to mean “overwrite the stored hash,” not “validate whether the source changed.”

---

## Topics to Explore

- [function] `reasons/check_stale.py:resolve_source_path` — Explains how `source` strings map to actual files using agent repos, `db_dir`, `repos`, and the `~/git` fallback, which determines whether `hash_sources` can hash a node.
- [function] `reasons/check_stale.py:check_stale` — Shows how stored `source_hash` values are later compared and how truncated hashes and git-aware pinning are handled.
- [file] `reasons/network.py` — Defines the `Network` and node attributes (`source`, `source_hash`, `metadata`, `repos`) that `hash_sources` relies on.
- [general] `cli-hash-sources-call-site` — Find the CLI command that calls `hash_sources` to see whether it filters nodes and saves the network afterward.
- [file] `tests/test_check_stale.py` — Tests staleness hashing behavior and likely documents edge cases for missing or unresolvable sources.

---

## Beliefs

- `hash-sources-mutates-network-nodes-in-place` — `hash_sources` assigns each resolved node’s `source_hash` to the full SHA-256 digest before returning results.
- `hash-sources-skips-existing-hashes-unless-forced` — With `force=False`, any node whose `source_hash` is truthy is skipped; with `force=True`, all nodes with a truthy `source` and resolvable path are re-hashed.
- `hash-sources-does-not-filter-by-truth-value` — The function iterates over all `network.nodes` and never checks `truth_value`, so it can hash nodes that are not `IN`.
- `hash-sources-requires-caller-persistence` — The function only mutates the in-memory `Network`; it never saves the network, so changes are lost unless the caller persists them.
- `hash-sources-propagates-read-errors` — `hash_sources` has no `try/except` around `hash_file`, so unreadable or directory paths raise and may leave earlier node mutations in place.
