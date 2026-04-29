# File: tests/test_check_stale.py

**Date:** 2026-04-29
**Time:** 16:59

No worries — I can give a thorough explanation from the test file alone. The tests are quite revealing about the production code's contracts.

---

## Purpose

`tests/test_check_stale.py` tests the **staleness-checking subsystem** of `ftl-reasons`. In this TMS, beliefs (nodes) can be grounded in source files — actual documents on disk that a premise was derived from. This module verifies that the system can detect when those source files have changed (or disappeared) since a belief was recorded, making the belief potentially stale.

It covers four units from `reasons_lib.check_stale`: `hash_file`, `resolve_source_path`, `check_stale`, and `hash_sources`.

## Key Components

### `TestHashFile`
Tests the lowest-level primitive: computing a SHA-256 hash of a file's contents. Two properties are verified:
1. **Correctness** — the hash matches `hashlib.sha256` applied to the same bytes.
2. **Sensitivity** — changing the file content changes the hash (no accidental caching or stale reads).

### `TestResolveSourcePath`
Tests path resolution for the `source` field on nodes. Source paths use a `repo/path` convention (e.g., `"myrepo/source.md"`), where the first segment is a repo alias mapped to a filesystem root via a `repos` dict. Three cases:
- **Happy path** — `"myrepo/entry.md"` resolves to `tmp_path / "entry.md"` when `repos={"myrepo": tmp_path}`.
- **Missing file** — returns `None` (not an exception).
- **Empty source** — returns `None`.

### `TestCheckStale`
Tests the main orchestration function. `check_stale(net, repos=...)` walks every IN node that has both a `source` and a `source_hash`, re-hashes the file, and reports mismatches. Key behaviors:

| Scenario | Expected |
|---|---|
| File unchanged | Empty results list |
| File changed | Result with `reason="content_changed"`, old/new hashes |
| Node is OUT (retracted) | Skipped — no staleness report |
| Node has no `source_hash` | Skipped |
| Source file deleted | Result with `reason="source_deleted"`, `new_hash=None`, `source_path=None` |
| Multiple stale nodes | All reported |

The return type is a list of dicts, each containing at minimum: `node_id`, `old_hash`, `new_hash`, `reason`, and (for deletions) `source_path`.

### `TestHashSources`
Tests the **backfill** function. `hash_sources` computes and writes hashes *into* the network for nodes that don't have one yet. This is a mutation operation — it modifies `net.nodes[id].source_hash` in place. Behaviors:

- **Backfill** — nodes with empty `source_hash` get hashed; result includes `was_empty: True`.
- **Skip existing** — nodes with a non-empty hash are left alone.
- **Force mode** — `force=True` re-hashes everything; result includes `was_empty: False`.
- **Missing files** — silently skipped (no crash, no result entry).
- **No source** — nodes without a `source` field are skipped.

## Patterns

**Repo alias mapping.** Source paths aren't absolute filesystem paths — they use a `repo-name/relative-path` scheme, decoupled from the filesystem via the `repos` dict. This lets the same network be checked against different checkout locations.

**Return-list-of-dicts over exceptions.** Both `check_stale` and `hash_sources` return a list of result dicts rather than raising on problems. Missing files, deleted sources, and empty hashes are all represented as data, not errors. This makes batch processing straightforward.

**`tmp_path` fixture throughout.** Every test that touches the filesystem uses pytest's `tmp_path`, ensuring isolation and cleanup. Files are written, optionally mutated, then checked — a classic "arrange, act, assert" pattern with filesystem state as the variable.

**Separation of read vs. write operations.** `check_stale` is read-only (reports staleness without mutating). `hash_sources` is write (backfills hashes into the network). The test suite mirrors this split with separate test classes.

## Dependencies

**Imports:**
- `hashlib` — used directly in tests to compute expected SHA-256 values.
- `pathlib.Path` — imported but not used explicitly (pytest's `tmp_path` is already a `Path`).
- `pytest` — test framework; only the `tmp_path` fixture is used (no custom markers or parametrize).
- `reasons_lib.network.Network` — the in-memory belief graph. Tests call `add_node()` and `retract()`.
- `reasons_lib.check_stale` — the module under test: `check_stale`, `hash_file`, `hash_sources`, `resolve_source_path`.

**Imported by:** Nothing — this is a leaf test module.

## Flow

A typical `check_stale` test follows this sequence:

1. **Create files** on disk via `tmp_path`.
2. **Build a `Network`** and add nodes with `source="repo/file.md"` and `source_hash=<sha256 of original content>`.
3. **Optionally mutate** the file on disk or retract the node.
4. **Call `check_stale(net, repos={"repo": tmp_path})`**.
5. **Assert** on the returned list — length, node IDs, reason codes, hash values.

For `hash_sources`, the flow is similar but step 5 also asserts that `net.nodes[id].source_hash` was mutated in place.

## Invariants

- `check_stale` only examines nodes that are **IN** (not retracted) **and** have a non-empty `source_hash`. Both conditions must hold.
- `resolve_source_path` returns `None` for missing files rather than raising — callers must handle `None`.
- `hash_sources` without `force=True` never overwrites an existing hash — it's an additive backfill.
- Result dicts always contain `node_id` and `reason` as identifying/classifying fields.
- Source paths follow the `repo-alias/relative-path` convention; the first `/`-delimited segment is the repo key.

## Error Handling

The staleness subsystem is conspicuously exception-free. Missing files, empty sources, retracted nodes — all handled via early returns or `None` values, never exceptions. This is deliberate: staleness checking is a batch audit operation that should report *all* problems, not abort on the first one.

The one implicit contract: if `repos` doesn't contain a key matching the source's repo prefix, `resolve_source_path` presumably returns `None`, and the node is either skipped or reported as `source_deleted`. The tests don't exercise an unknown-repo scenario directly, but the `source_deleted` test covers the analogous case (file not found after resolution).

## Topics to Explore

- [file] `reasons_lib/check_stale.py` — The production implementation; see how repo aliases are split, how `force` is threaded through, and whether there's any logging or side-effect beyond hash mutation
- [function] `reasons_lib/network.py:Network.add_node` — Understand the full signature, especially how `source` and `source_hash` are stored as node attributes
- [file] `tests/test_check_stale_issue25.py` — A regression test file for a specific issue; likely covers an edge case that the main test file doesn't
- [general] `source-path-convention` — How the `repo/relative-path` scheme is used across the codebase (CLI, import, export) and whether repo aliases are configured globally or per-invocation
- [function] `reasons_lib/network.py:Network.retract` — How retraction sets a node to OUT, since `check_stale` relies on this status to skip retracted nodes

## Beliefs

- `check-stale-skips-out-nodes` — `check_stale` only reports staleness for IN nodes; retracted (OUT) nodes are silently skipped even if their source file has changed
- `hash-sources-is-additive-by-default` — `hash_sources` without `force=True` never overwrites an existing non-empty `source_hash`; it only backfills empty ones
- `resolve-source-path-returns-none-on-missing` — `resolve_source_path` returns `None` (not an exception) when the resolved file does not exist on disk
- `staleness-results-are-dicts-not-exceptions` — Both `check_stale` and `hash_sources` report problems as list-of-dict return values, never raising exceptions for missing/changed files
- `source-paths-use-repo-alias-prefix` — Node source paths follow a `repo-alias/relative-path` convention where the first path segment is a key into the `repos` dict mapping aliases to filesystem roots

