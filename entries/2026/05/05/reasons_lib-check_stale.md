# File: reasons_lib/check_stale.py

**Date:** 2026-05-05
**Time:** 15:20

## Purpose

`check_stale.py` is the staleness detection subsystem for the reasons belief network. It answers the question: *has the source material a belief was derived from changed since the belief was recorded?*

Every node in the belief network can carry a `source` (a file path like `entries/2026/04/15/exploration.md`) and a `source_hash` (SHA-256 of that file's content at the time the belief was created). This module compares stored hashes against current file contents to find beliefs that may need re-evaluation.

It owns three responsibilities:
1. **Staleness detection** — finding nodes whose source files changed or disappeared
2. **Hash backfilling** — adding hashes to nodes that have sources but were created before hashing was implemented
3. **Hash format migration** — upgrading truncated 16-char hash prefixes to full 64-char SHA-256 hashes

## Key Components

### `hash_file(path: Path) -> str`
Pure function. Reads a file and returns its full SHA-256 hex digest (64 characters). No error handling — will raise on unreadable files.

### `resolve_source_path(source, repos, db_dir, agent) -> Path | None`
Resolves a relative source string to an absolute filesystem path. Uses a priority chain:

1. **Agent repo** — if the node was imported by an agent, look in that agent's repo directory first
2. **db_dir** — the directory containing `reasons.db`, for expert repos where entries live alongside the database
3. **repos dict** — uses the first path component as a repo name key (e.g., `ftl-reasons/src/foo.py` → `repos["ftl-reasons"] / "src/foo.py"`)
4. **`~/git/` fallback** — convention-based: `~/git/{repo_name}/{rel_path}`
5. **Bare path** — for single-component paths with no `/`, tries it as-is

Returns `None` if no resolution finds an existing file — never raises.

### `check_stale(network, repos, db_dir, upgrade_hashes) -> (list[dict], int)`
The main staleness check. Iterates all IN nodes that have both `source` and `source_hash`. For each node, it classifies the result into one of three categories:

| `reason` | Meaning |
|---|---|
| `"source_deleted"` | The source file can't be found at any resolution path |
| `"truncated_hash"` | The stored hash is a 16-char prefix of the current full hash — a legacy format, not actual content change |
| `"content_changed"` | The file exists but its content hash differs from what was stored |

When `upgrade_hashes=True`, truncated hashes that match the current file's hash prefix are silently upgraded in-place on the node object (caller must persist). These are excluded from the stale results.

### `hash_sources(network, repos, force, db_dir) -> list[dict]`
Backfill operation. Finds nodes with a `source` but no `source_hash` and computes the hash. With `force=True`, re-hashes all source-bearing nodes regardless of existing hash — useful after confirming a source change is intentional and updating the baseline.

Mutates `node.source_hash` directly; caller must save the network.

## Patterns

**Resolution chain with fallback** — `resolve_source_path` implements a priority-ordered search across multiple root directories, degrading gracefully to `None`. This is a common pattern for tools that operate across multiple repos or installations.

**Mutation-then-caller-saves** — Both `check_stale` (with `upgrade_hashes`) and `hash_sources` mutate `node.source_hash` directly on the `Network` object but don't persist. The caller is responsible for saving. This keeps I/O concerns out of the detection logic.

**Structured result dicts** — Results are plain dicts with a `reason` discriminator field rather than typed objects. This keeps the module lightweight and makes the results easy to serialize for CLI output.

**Sorted iteration** — Both `check_stale` and `hash_sources` iterate `sorted(network.nodes.items())`, producing deterministic output regardless of dict insertion order.

## Dependencies

**Imports:**
- `hashlib` — SHA-256 hashing
- `pathlib.Path` — filesystem operations
- `.network.Network` — the in-memory belief graph that holds nodes, their truth values, sources, and hashes

**Imported by:**
- `reasons_lib/api.py` — the public API layer that exposes `check-stale` and `hash-sources` as CLI commands
- `tests/test_check_stale.py` and `tests/test_check_stale_issue25.py` — unit and regression tests

## Flow

For `check_stale`:

```
network.nodes ──filter(IN, has source+hash)──▶ resolve_source_path
                                                    │
                                    ┌───────────────┼───────────────┐
                                    ▼               ▼               ▼
                               None returned    hash matches    hash differs
                                    │               │               │
                             source_deleted      (skip)        is it a 16-char
                                                               prefix match?
                                                                 /     \
                                                              yes       no
                                                               │         │
                                                         truncated   content_changed
                                                         _hash       
                                                         (or upgrade
                                                          if flag set)
```

For `hash_sources`: simpler linear flow — resolve path, compute hash, write it onto the node.

## Invariants

- Only `IN` nodes are checked for staleness — OUT nodes are already not believed, so their source currency is irrelevant.
- A node must have **both** `source` and `source_hash` to be stale-checkable. Missing either means it's skipped (not flagged as stale).
- Truncated hashes are exactly 16 characters. This is the sole distinguishing criterion between a legacy short hash and a hash that simply doesn't match. A 16-char stored hash that is a prefix of the current full hash is treated as "same content, old format" — not as stale.
- `resolve_source_path` returns `None` rather than raising for missing files. The caller distinguishes "can't find file" from "file changed."
- The `repos` dict auto-populates from `network.repos` if not provided explicitly, converting string values to `Path` objects.

## Error Handling

This module is notably quiet on errors:

- **File not found**: `resolve_source_path` returns `None`; `check_stale` records it as `"source_deleted"`. No exception.
- **File unreadable**: `hash_file` will raise (likely `PermissionError` or `OSError`) — this is **not** caught. A single unreadable file will abort the entire check.
- **No source/hash**: Silently skipped via `continue`.
- **`hash_sources` with missing file**: Skips the node (path resolves to `None`), no error recorded. The caller gets no indication the node was skipped.

## Topics to Explore

- [file] `reasons_lib/network.py` — Defines the `Network` and node model that this module operates on; understanding `source`, `source_hash`, `truth_value`, and `metadata` fields is essential
- [file] `reasons_lib/api.py` — The API layer that calls `check_stale` and `hash_sources`, showing how results are surfaced to users and how persistence is handled after mutations
- [file] `tests/test_check_stale_issue25.py` — Regression test for a specific bug; reveals edge cases in the staleness logic that drove design decisions
- [function] `reasons_lib/check_stale.py:resolve_source_path` — The resolution priority chain is the most subtle part of this module; trace it with multi-repo and agent-imported scenarios
- [general] `hash-upgrade-migration` — The truncated-hash detection (16-char prefix check) implies a historical format change; understanding when and why short hashes were used clarifies why this migration path exists

## Beliefs

- `check-stale-only-examines-in-nodes` — `check_stale` skips all nodes with `truth_value != "IN"`; OUT and undecided nodes are never reported as stale
- `resolve-source-path-never-raises` — `resolve_source_path` returns `None` for all failure modes (missing file, empty source, no matching repo) instead of raising exceptions
- `truncated-hash-threshold-is-16-chars` — A stored `source_hash` of exactly 16 characters that is a prefix of the current full hash is classified as `truncated_hash`, not `content_changed`
- `check-stale-and-hash-sources-mutate-in-place` — Both functions modify `node.source_hash` directly on the `Network` object; neither persists the change — the caller must save

