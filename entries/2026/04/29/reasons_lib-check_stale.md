# File: reasons_lib/check_stale.py

**Date:** 2026-04-29
**Time:** 16:58

# `reasons_lib/check_stale.py` — Source Staleness Detection

## Purpose

This module answers one question: **has the source file a belief was derived from changed since the belief was recorded?**

In a Truth Maintenance System, beliefs (nodes) can be sourced from files — code, markdown, documentation. When those files change on disk, any belief derived from them may no longer be valid. This module provides the machinery to detect that drift and to backfill hashes for nodes that were created before hash-tracking existed.

It owns two responsibilities:
1. **Staleness checking** — compare stored SHA-256 hashes against current file content
2. **Hash backfilling** — populate `source_hash` on nodes that have a `source` path but no stored hash

## Key Components

### `hash_file(path: Path) -> str`
Computes the full SHA-256 hex digest (64 chars) of a file. Pure function, no caching. Reads the entire file into memory via `read_bytes()`.

### `resolve_source_path(source: str, repos: dict[str, Path] | None) -> Path | None`
Translates a source string like `"ftl-reasons/src/foo.py"` into an absolute filesystem path. Resolution strategy:
1. If `repos` mapping is provided and the repo name matches, use that as the base directory.
2. Otherwise, fall back to `~/git/<repo-name>/<rel-path>`.
3. If the source has no slash, treat it as a bare filename in the current directory.
4. Returns `None` if the resolved path doesn't exist — this is the "source deleted" signal.

### `check_stale(network, repos) -> list[dict]`
The main staleness check. Iterates all nodes in the network and returns a list of problems. Each result is a dict with one of two `reason` values:
- **`content_changed`** — the file exists but its hash differs from the stored one
- **`source_deleted`** — the file no longer exists at the resolved path

Only examines nodes where `truth_value == "IN"` and both `source` and `source_hash` are set. OUT nodes are irrelevant (they're already retracted), and nodes without sources aren't file-backed.

### `hash_sources(network, repos, force) -> list[dict]`
Backfill operation. Walks all nodes (regardless of truth value) that have a `source` but no `source_hash`, resolves the path, computes the hash, and writes it directly onto `node.source_hash`. With `force=True`, re-hashes even nodes that already have a hash — useful after confirming a source change is intentional and "accepting" the new state.

Returns metadata about what was hashed, including `was_empty` to distinguish initial backfill from forced re-hash.

## Patterns

**Convention-based path resolution** — The `~/git/<repo-name>` fallback is a workspace convention, not configuration. The `repos` override parameter lets callers specify explicit mappings when the convention doesn't hold (tests, CI, alternate checkouts).

**Report-don't-act** — Both `check_stale` and `hash_sources` return structured dicts rather than taking action. The caller decides what to do with stale nodes (retract them, warn the user, etc.). `hash_sources` is the exception — it mutates `node.source_hash` in place — but it still returns a report of what changed.

**Sorted iteration** — Both functions iterate `sorted(network.nodes.items())`, giving deterministic output order. This matters for stable diffs in exported belief files.

## Dependencies

**Imports:**
- `hashlib` — SHA-256 computation
- `pathlib.Path` — filesystem path resolution and existence checks
- `.network.Network` — the in-memory belief graph; provides `nodes` dict mapping node IDs to node objects with `truth_value`, `source`, and `source_hash` attributes

**Imported by:**
- `reasons_lib/api.py` — exposes `check_stale` and `hash_sources` as API commands
- `tests/test_check_stale.py` and `tests/test_check_stale_issue25.py` — unit tests

## Flow

**`check_stale` control flow:**
```
for each node (sorted by ID):
    skip if OUT, or missing source/source_hash
    resolve source path
    ├── path is None → emit "source_deleted" record
    └── path exists → hash file
        ├── hash matches → skip (node is fresh)
        └── hash differs → emit "content_changed" record
```

**`hash_sources` control flow:**
```
for each node (sorted by ID):
    skip if no source
    skip if source_hash already set AND force=False
    resolve source path → skip if None
    compute hash → write to node.source_hash
    emit record with was_empty flag
```

## Invariants

1. **Only IN nodes are checked for staleness.** OUT nodes are excluded by the `truth_value != "IN"` guard. This prevents false positives on already-retracted beliefs.
2. **Nodes without both `source` and `source_hash` are silently skipped** — no error, no report entry. This is intentional: many nodes are manually entered premises with no file backing.
3. **`resolve_source_path` returns `None` for nonexistent paths** — it never raises. Callers must check for `None`.
4. **`hash_sources` mutates the network in place** while `check_stale` is read-only. This asymmetry is important: staleness checking is safe to run repeatedly, but hash backfilling changes state.
5. **Hash comparison is exact string equality** on 64-char hex digests. No prefix matching or fuzzy comparison.

## Error Handling

Minimal and deliberate. The module relies on `Path.exists()` to avoid `FileNotFoundError` and returns `None` from `resolve_source_path` rather than raising. `Path.read_bytes()` in `hash_file` will raise if the file is unreadable (permissions, broken symlink), but this is not caught — it propagates to the caller. There's no retry logic or partial-failure handling; a single unreadable file will abort the entire check.

---

## Topics to Explore

- [file] `reasons_lib/network.py` — Defines the `Network` class and node model that `check_stale` operates on; understanding `source`, `source_hash`, and `truth_value` attributes is essential
- [file] `tests/test_check_stale.py` — Shows the expected behavior for content-changed, source-deleted, and hash-backfill scenarios with concrete fixtures
- [file] `tests/test_check_stale_issue25.py` — Regression test for a specific staleness bug; reveals an edge case that drove a code change
- [function] `reasons_lib/api.py:check_stale` — The API layer that calls into this module; shows how staleness results are surfaced to the CLI and whether any automatic retraction happens
- [general] `source-hash-lifecycle` — How `source_hash` is initially set (during import? during `hash_sources`?), updated, and used across the retract/re-derive cycle

## Beliefs

- `check-stale-skips-out-nodes` — `check_stale` only examines nodes with `truth_value == "IN"`; OUT nodes are never reported as stale
- `resolve-source-path-never-raises` — `resolve_source_path` returns `None` for missing files instead of raising exceptions; callers must handle the `None` case
- `hash-sources-mutates-network` — `hash_sources` writes directly to `node.source_hash` on the in-memory network, while `check_stale` is purely read-only
- `staleness-uses-full-sha256` — Staleness detection compares full 64-character SHA-256 hex digests, not truncated prefixes
- `resolve-source-defaults-to-home-git` — When no `repos` mapping is provided, `resolve_source_path` falls back to `~/git/<repo-name>/<rel-path>` by convention

