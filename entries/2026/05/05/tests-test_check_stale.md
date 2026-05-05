# File: tests/test_check_stale.py

**Date:** 2026-05-05
**Time:** 15:21

## Purpose

`tests/test_check_stale.py` is the test suite for the **staleness detection subsystem** (`reasons_lib/check_stale.py`). Its job is to verify that the system can detect when a belief's source material (typically a markdown file on disk) has changed since the belief was recorded, allowing the TMS to flag beliefs that may need re-evaluation.

This is a critical integrity mechanism: beliefs in the network are derived from source documents. If a source document changes but the belief doesn't get reviewed, the knowledge base silently drifts from its evidence. This module prevents that.

## Key Components

### `TestHashFile`
Validates `hash_file()` — a thin wrapper around SHA-256 that hashes file contents. Two tests confirm it produces the expected digest and that different content produces different hashes. Pure utility, no interesting logic.

### `TestResolveSourcePath`
Validates `resolve_source_path(source, repos=None, db_dir=None, agent=None)` — the path resolution logic that maps a belief's `source` string (e.g. `"entries/2026/topic.md"`) to an actual file on disk. This is the most complex unit because source strings can resolve through multiple strategies:

1. **`db_dir` lookup** — resolve relative to the expert knowledge base directory (highest priority)
2. **`agent` repo lookup** — resolve relative to the agent's registered repo directory
3. **`repos` dict lookup** — split the source on `/`, use the first segment as a repo key, join the rest as a relative path
4. **Fallback ordering** — `db_dir` > agent repo > repo-split

The tests establish the **precedence chain**: `db_dir` beats repos (`test_db_dir_takes_precedence`), agent repo beats repo-split (`test_agent_repo_takes_precedence_over_split`), and missing files return `None`.

### `TestCheckStale`
Validates `check_stale(net, repos=None, db_dir=None)` — the main entry point that iterates over all IN nodes with source hashes, resolves their source files, re-hashes, and reports mismatches. Returns `(results, upgraded_count)`.

Key behaviors tested:
- **Fresh node** — hash matches, empty results list
- **Stale node** — hash mismatch, result with `reason="content_changed"`
- **Deleted source** — file gone, result with `reason="source_deleted"` and `new_hash=None`
- **OUT nodes skipped** — retracted beliefs are not checked (no point flagging stale evidence for beliefs already retracted)
- **No hash skipped** — nodes without a `source_hash` are silently ignored
- **Multi-agent resolution** — nodes with `metadata.agent` resolve through `net.repos` using the agent lookup path

### `TestPrefixHashUpgrade`
Handles a migration concern: early versions of the system stored **truncated 16-char hash prefixes** instead of full SHA-256 digests. This test class validates the upgrade path:

- With `upgrade_hashes=True`: if the truncated hash is a prefix of the current full hash, the node is treated as fresh and its hash is silently upgraded in-place. Returns the upgrade count.
- Without the flag: truncated hashes produce a `reason="truncated_hash"` warning result instead.
- Genuine content changes are still detected even with truncated hashes and the upgrade flag on.

### `TestHashSources`
Validates `hash_sources(net, repos=None, db_dir=None, force=False)` — a backfill utility that computes and stores hashes for nodes that don't have one yet. Behaviors:

- Backfills nodes with empty `source_hash`
- Skips nodes that already have a hash (unless `force=True`)
- Skips nodes with missing source files or no `source` at all
- Returns a list of `{node_id, was_empty}` dicts describing what was touched

## Patterns

**Filesystem-isolated tests via `tmp_path`** — every test uses pytest's `tmp_path` fixture to create an isolated directory tree. No test touches the real filesystem or the real `reasons.db`.

**Network as in-memory fixture** — tests instantiate `Network()` directly (no database, no fixtures beyond `tmp_path`), add nodes with `add_node()`, and pass the network to the functions under test. This keeps tests fast and dependency-free.

**Result-as-dict convention** — stale results are plain dicts with keys `node_id`, `old_hash`, `new_hash`, `reason`, `source_path`. No result objects or exceptions — the caller inspects the list.

**Tuple return** — `check_stale` returns `(results, upgraded_count)`, which tests destructure with `results, _ = check_stale(...)` when they don't care about the upgrade count.

## Dependencies

**Imports:**
- `hashlib` — SHA-256 computation for expected values in assertions
- `pathlib.Path` — imported but not used directly (likely left from a refactor; `tmp_path` provides `Path` objects)
- `pytest` — imported but no explicit `pytest.mark` or `pytest.raises` usage; present for fixture support
- `reasons_lib.network.Network` — the in-memory belief graph
- `reasons_lib.check_stale` — the module under test (`check_stale`, `hash_file`, `hash_sources`, `resolve_source_path`)

**Imported by:** Nothing — this is a leaf test module.

## Flow

A typical staleness check follows this path:

1. A `Network` is populated with nodes that have `source` paths and `source_hash` digests
2. `check_stale()` iterates IN nodes, calls `resolve_source_path()` to find the file, calls `hash_file()` to compute the current hash
3. If hashes mismatch → append `{reason: "content_changed"}` to results
4. If file missing → append `{reason: "source_deleted"}`
5. If truncated hash detected → either upgrade in-place or append `{reason: "truncated_hash"}`
6. Return `(stale_results, upgrade_count)`

`hash_sources()` follows a parallel path but writes hashes *into* nodes rather than comparing them.

## Invariants

- **OUT nodes are never checked for staleness** — only IN nodes matter
- **Nodes without `source_hash` are silently skipped** by `check_stale` (but are candidates for `hash_sources`)
- **`db_dir` resolution takes precedence** over repo-split resolution
- **Agent repo resolution takes precedence** over repo-split for agent-imported nodes
- **Truncated hash upgrade is opt-in** — without `upgrade_hashes=True`, truncated hashes are reported as warnings, not silently fixed
- **`hash_sources` is idempotent** without `force` — nodes with existing hashes are untouched

## Error Handling

There is essentially none in these tests — and that's intentional. The functions under test return result lists rather than raising exceptions. Missing files produce `None` from `resolve_source_path` and `reason="source_deleted"` results from `check_stale`. There are no `pytest.raises` calls anywhere, confirming the design philosophy: staleness is a reportable condition, not an error.

## Topics to Explore

- [file] `reasons_lib/check_stale.py` — The implementation these tests cover; see how `resolve_source_path` handles the precedence chain and how `check_stale` iterates nodes
- [file] `reasons_lib/network.py` — The `Network` class and `Node` dataclass, particularly `add_node()`, `retract()`, `source`, `source_hash`, and `repos` dict
- [file] `tests/test_check_stale_issue25.py` — A companion test file likely covering a specific bug fix; compare coverage with this file
- [function] `reasons_lib/check_stale.py:resolve_source_path` — The multi-strategy path resolution logic with db_dir/agent/repos fallback ordering
- [general] `truncated-hash-migration` — Understand why 16-char prefix hashes existed and how the upgrade path works across the CLI and storage layers

## Beliefs

- `check-stale-skips-out-nodes` — `check_stale` only examines nodes whose truth value is IN; retracted (OUT) nodes are excluded from staleness checks
- `resolve-source-path-db-dir-precedence` — When both `db_dir` and `repos` can resolve a source path, `db_dir` takes precedence
- `check-stale-returns-dicts-not-exceptions` — Staleness conditions (content changed, source deleted, truncated hash) are reported as result dicts, never raised as exceptions
- `hash-sources-idempotent-without-force` — `hash_sources` skips nodes that already have a non-empty `source_hash` unless `force=True`
- `truncated-hash-upgrade-opt-in` — Prefix hash upgrade only occurs when `upgrade_hashes=True` is passed; without it, truncated hashes produce a warning result with `reason="truncated_hash"`

