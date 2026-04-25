# File: reasons_lib/check_stale.py

**Date:** 2026-04-24
**Time:** 16:58



# `reasons_lib/check_stale.py` — Source Staleness Detection

## Purpose

This module answers one question: **has the real-world source that a belief was derived from changed since the belief was recorded?**

In a truth maintenance system, beliefs aren't just logical propositions — they can be grounded in external artifacts (files, documents). If the source file changes, the belief may no longer be valid. This module bridges the gap between the TMS's internal dependency graph and the filesystem, providing the mechanism to detect when beliefs need re-examination.

It owns two responsibilities:
1. **Staleness detection** (`check_stale`) — scan all IN nodes, compare stored hashes against current file contents
2. **Hash backfilling** (`hash_sources`) — populate or refresh source hashes for nodes that have source paths but no stored hash

## Key Components

### `hash_file(path: Path) -> str`
Pure utility — SHA-256 of file contents, returned as a 64-char hex string. No error handling; will raise if the file can't be read. This is intentional — callers gate on `path.exists()` before reaching this point.

### `resolve_source_path(source: str, repos: dict[str, Path] | None) -> Path | None`
Translates a source string (stored on a node) into a filesystem path. The source format is `repo-name/relative/path.md`. Resolution strategy:

1. If `repos` dict is provided and contains the repo name → use that mapping
2. Otherwise → fall back to `~/git/<repo-name>/relative/path.md`
3. If the source has no slash → treat as a bare filename in the current directory
4. Returns `None` if the resolved path doesn't exist on disk

This convention ties into a multi-repo workflow where beliefs can reference files across different repositories.

### `check_stale(network, repos) -> list[dict]`
The main detection function. Iterates all nodes in sorted order and checks each IN node that has both a `source` and `source_hash`. Returns a list of result dicts, each categorized as either:

- **`content_changed`** — file exists but hash differs from stored value
- **`source_deleted`** — file no longer exists at the resolved path

Nodes that are OUT, or lack source/source_hash metadata, are silently skipped.

### `hash_sources(network, repos, force) -> list[dict]`
The complement to `check_stale` — writes hashes rather than reading them. Two modes:

- Default (`force=False`): only fills in missing hashes (backfill)
- `force=True`: re-hashes everything, useful after acknowledging a known source change

Mutates nodes in-place (`node.source_hash = new_hash`) and returns a manifest of what was updated, including a `was_empty` flag to distinguish backfills from overwrites.

## Patterns

**Report-don't-act**: Both functions return structured dicts rather than performing any retraction or network mutation (beyond hash backfill). The caller (typically `api.py`) decides what to do with staleness information. This keeps the module focused on detection, not policy.

**Sorted iteration**: `sorted(network.nodes.items())` ensures deterministic output ordering — important for reproducible test assertions and predictable CLI output.

**Convention over configuration**: The `~/git/<repo-name>` fallback is a sensible default for local development, while the `repos` parameter allows explicit overrides for different environments.

**None-as-absence**: `resolve_source_path` returns `None` for unresolvable paths rather than raising, enabling the caller to distinguish "source deleted" from "source changed" cleanly.

## Dependencies

**Imports:**
- `hashlib` — SHA-256 computation
- `pathlib.Path` — filesystem operations
- `.network.Network` — the belief graph; provides `.nodes` dict of `Node` objects

**Imported by:**
- `reasons_lib/api.py` — exposes `check_stale` and `hash_sources` as part of the functional API
- `tests/test_check_stale.py` and `tests/test_check_stale_issue25.py` — unit tests

The module depends only on `Network` for reading node data. It does not import `storage.py` — persistence of updated hashes is the caller's responsibility.

## Flow

For `check_stale`:
```
iterate sorted nodes
  → skip OUT nodes
  → skip nodes without source or source_hash
  → resolve source path
    → None? → emit "source_deleted" record
    → exists? → hash file → compare with stored hash
      → mismatch? → emit "content_changed" record
      → match? → skip (fresh)
```

For `hash_sources`:
```
iterate sorted nodes
  → skip nodes without source
  → skip nodes with existing hash (unless force=True)
  → resolve source path
    → None? → skip silently
    → exists? → hash file → mutate node.source_hash → emit record
```

## Invariants

- Only **IN** nodes are checked for staleness — OUT nodes are already considered non-believed
- A node must have **both** `source` and `source_hash` to be eligible for stale-checking
- `hash_file` always produces a full 64-char SHA-256 hex string (per the commit `3c400d4` which moved from truncated 16-char hashes to full hashes)
- `hash_sources` with `force=False` never overwrites an existing hash
- Result lists preserve sorted node ID order

## Error Handling

Minimal by design. `resolve_source_path` uses existence checks (`p.exists()`) to avoid raising on missing files, returning `None` instead. `hash_file` does no error handling — if a file exists but can't be read (permissions, race condition), the exception propagates to the caller. This is a reasonable tradeoff: if you can see the file but can't read it, that's an environmental problem worth surfacing.

---

## Topics to Explore

- [file] `reasons_lib/api.py` — Where `check_stale` and `hash_sources` are called and how staleness results get translated into CLI output or network mutations
- [file] `tests/test_check_stale_issue25.py` — Regression test likely related to a specific staleness bug; understanding the edge case it covers
- [function] `reasons_lib/network.py:Node` — The `source` and `source_hash` fields on Node that this module reads and writes
- [diff] `3c400d4` — The commit that switched from truncated 16-char hashes to full SHA-256, which directly shaped `hash_file`'s contract
- [general] `staleness-vs-retraction` — How staleness detection feeds into the TMS retraction cascade — does detecting staleness automatically retract, or is it a manual step?

## Beliefs

- `check-stale-skips-out-nodes` — `check_stale` only examines nodes with `truth_value == "IN"`; OUT nodes are never reported as stale
- `hash-sources-no-overwrite-default` — `hash_sources` with `force=False` will never modify a node that already has a `source_hash` value
- `resolve-source-fallback-path` — When no `repos` mapping is provided, `resolve_source_path` constructs paths as `~/git/<repo-name>/<relative-path>`
- `hash-file-full-sha256` — `hash_file` returns a 64-character hex SHA-256 digest, not a truncated hash
- `check-stale-report-only` — `check_stale` never mutates the network; it returns detection results for the caller to act on

