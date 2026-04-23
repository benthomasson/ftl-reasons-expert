# Function: check_stale in reasons_lib/check_stale.py

**Date:** 2026-04-23
**Time:** 16:48

## `check_stale` — Staleness Detector for Belief Network Nodes

### 1. Purpose

`check_stale` answers the question: *"Have any of the source files that my currently-held beliefs are based on changed since I recorded those beliefs?"*

In the `ftl-reasons` truth maintenance system, nodes (beliefs) can be grounded in source files — e.g., a belief about how `router.py` works has `source="myrepo/src/router.py"` and a stored SHA-256 hash of that file's content at the time the belief was created. When the source file changes, the belief may no longer be accurate. This function detects that drift.

It exists to support the `reasons check-stale` workflow, giving the operator a list of beliefs that need human review because their evidentiary basis has changed on disk.

### 2. Contract

**Preconditions:**
- `network` must be a populated `Network` instance with nodes that have `source` and `source_hash` fields set.
- If `repos` is provided, its values must be `Path` objects pointing to existing directories.

**Postconditions:**
- Returns a list of dicts describing every stale node found.
- The `network` is **not mutated** — this is a read-only scan.
- All returned entries correspond to nodes that are currently `IN`, have both `source` and `source_hash` set, and whose source file exists on disk with a different hash.

**Invariant:** If an empty list is returned, either (a) no IN nodes have source tracking, (b) all source files match their stored hashes, or (c) all source files are missing from disk.

### 3. Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `network` | `Network` | The belief network to scan. Only its `.nodes` dict is read. |
| `repos` | `dict[str, Path] \| None` | Optional mapping from repo short names (e.g., `"myrepo"`) to filesystem paths. When `None`, `resolve_source_path` falls back to `~/git/<repo-name>/...`. |

**Edge cases for `repos`:**
- If a repo name in a node's `source` isn't in the `repos` dict, the fallback `~/git/<name>` path is tried. If that doesn't exist either, the node is silently skipped.
- If `repos` maps a name to a directory that exists but the specific file within it doesn't, the node is skipped.

### 4. Return Value

Returns `list[dict]`, where each dict has:

| Key | Type | Meaning |
|-----|------|---------|
| `node_id` | `str` | The ID of the stale node |
| `old_hash` | `str` | The hash stored on the node (from when the belief was created) |
| `new_hash` | `str` | The current hash of the source file |
| `source` | `str` | The node's source string (e.g., `"myrepo/path/file.py"`) |
| `source_path` | `str` | The resolved absolute path to the source file |

An empty list means "nothing is stale (or nothing is checkable)." The caller cannot distinguish "all fresh" from "no nodes have source tracking" without inspecting the network separately.

### 5. Algorithm

1. **Iterate nodes in sorted order** — `sorted(network.nodes.items())` gives deterministic output regardless of insertion order.
2. **Filter to candidates** — skip any node where:
   - `truth_value != "IN"` — retracted beliefs don't need freshness checks
   - `source` is falsy — no source file linked
   - `source_hash` is falsy — hash was never recorded (use `hash_sources` to backfill)
3. **Resolve the source path** — delegates to `resolve_source_path`, which splits the source string on the first `/` to extract a repo name, looks it up in `repos` or falls back to `~/git/<repo>/`, and returns `None` if the file doesn't exist.
4. **Hash the current file** — computes SHA-256 of the file contents, truncated to 16 hex characters.
5. **Compare** — if `current_hash != node.source_hash`, the node is stale. Append a result dict.
6. **Return** the accumulated results.

### 6. Side Effects

**None.** This function is purely read-only with respect to the network. It does perform filesystem I/O (reading source files to hash them), but it does not modify any node's state, truth value, or metadata.

This is a deliberate design choice — staleness detection is separated from staleness resolution. The caller decides what to do with the results (retract, re-verify, update hashes, etc.).

### 7. Error Handling

The function swallows most error conditions silently:

- **Missing source file** → `resolve_source_path` returns `None` → node is skipped. No error reported. This means you can't distinguish "file was deleted" from "file was never there."
- **Empty source or hash** → skipped via the `continue` guards.
- **File read errors** — `hash_file` calls `path.read_bytes()`, which will raise `OSError`/`PermissionError` if the file exists but can't be read. This is **not caught** and will propagate to the caller, aborting the scan partway through.

The function does **not** raise on an empty network or a network with no IN nodes — it just returns `[]`.

### 8. Usage Patterns

Typical call from the CLI (`reasons check-stale`):

```python
stale = check_stale(network, repos={"ftl-reasons": Path("/Users/ben/git/ftl-reasons")})
for s in stale:
    print(f"⚠ {s['node_id']} is stale — {s['source']} changed")
    # Then: retract, update hash, or prompt user
```

**Caller obligations:**
- Must have a loaded `Network` — typically from `reasons.db` via the persistence layer.
- Should handle the "stale" results: either retract stale nodes, update their hashes (via `hash_sources(..., force=True)`), or flag them for human review.
- Should be prepared for `OSError` from unreadable files.

### 9. Dependencies

| Dependency | Role |
|------------|------|
| `reasons_lib.network.Network` | Provides the node graph to scan |
| `reasons_lib.Node` | Node data structure (accessed via `network.nodes`) — fields: `truth_value`, `source`, `source_hash` |
| `resolve_source_path` (same module) | Maps `"repo/path"` strings to absolute `Path` objects |
| `hash_file` (same module) | SHA-256 truncated to 16 hex chars |
| `hashlib` (stdlib) | Underlying hash implementation |
| `pathlib.Path` (stdlib) | Filesystem operations |

No external packages. No database access. No network calls.

---

## Topics to Explore

- [function] `reasons_lib/check_stale.py:hash_sources` — The complementary function that backfills or updates `source_hash` on nodes; used after confirming a stale source change is expected
- [function] `reasons_lib/network.py:retract` — How retraction cascades work when a stale belief is actually removed from the belief set
- [function] `reasons_lib/check_stale.py:resolve_source_path` — The path resolution logic and its `~/git/<name>` fallback convention, which is an implicit contract with the user's filesystem layout
- [file] `reasons_lib/network.py` — The full Network class, including truth propagation and the TMS algorithm that determines what goes IN/OUT
- [general] `staleness-resolution-workflow` — How the CLI orchestrates check-stale → user review → retract/re-hash, and what happens to downstream derived beliefs when a source-backed premise is retracted

## Beliefs

- `check-stale-is-read-only` — `check_stale` never mutates the network; it returns a report and leaves all nodes unchanged
- `check-stale-skips-out-nodes` — Only nodes with `truth_value == "IN"` are checked; retracted (OUT) nodes are ignored even if their source has changed
- `check-stale-requires-both-source-fields` — A node must have both `source` (non-empty) and `source_hash` (non-empty) to be eligible for staleness checking; nodes missing either are silently skipped
- `hash-truncation-is-16-hex` — Source hashes are SHA-256 truncated to the first 16 hex characters (64 bits), which means collision resistance is reduced from 128 bits to 32 bits for birthday attacks
- `missing-source-file-is-silent` — If a source file no longer exists on disk, the node is skipped without any indication; callers cannot distinguish "file deleted" from "file never tracked"

