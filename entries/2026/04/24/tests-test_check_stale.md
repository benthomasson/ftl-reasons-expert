# File: tests/test_check_stale.py

**Date:** 2026-04-24
**Time:** 16:58

# `tests/test_check_stale.py` — Staleness Detection Tests

## Purpose

This file tests the staleness detection subsystem: the mechanism that detects when a belief's source material has changed on disk since the belief was recorded. In a TMS, beliefs derived from external sources (files, documents) can become stale when those sources are modified or deleted. This test suite validates that the system correctly identifies those cases and distinguishes them from fresh, unchanged beliefs.

It covers four functions exported from `reasons_lib.check_stale`: `hash_file`, `resolve_source_path`, `check_stale`, and `hash_sources`.

## Key Components

### `TestHashFile`
Tests the lowest-level primitive — SHA-256 hashing of a file's contents. Two properties verified:
1. The hash matches a manually computed `hashlib.sha256` digest (correctness).
2. Changing the file content produces a different hash (sensitivity).

### `TestResolveSourcePath`
Tests source path resolution via a `repos` dictionary that maps logical repo names to filesystem paths. Source references use the format `"reponame/relative/path"`. Validates:
- A valid repo prefix + existing file resolves to a `Path`.
- A valid repo prefix + missing file returns `None`.
- An empty source string returns `None`.

### `TestCheckStale`
The core test class. `check_stale(net, repos)` scans all IN nodes that have a `source` and `source_hash`, re-hashes the source file, and reports mismatches. Tests cover:
- **Fresh node** — hash matches, returns empty list.
- **Stale node** — content changed, returns a dict with `node_id`, `old_hash`, `new_hash`, and `reason: "content_changed"`.
- **OUT nodes skipped** — retracted beliefs aren't checked (no point flagging staleness on something already disbelieved).
- **Missing hash skipped** — nodes without a recorded `source_hash` can't be compared.
- **Deleted source file** — returns `reason: "source_deleted"` with `new_hash: None` and `source_path: None`.
- **Multiple stale** — confirms batch detection across multiple nodes.

### `TestHashSources`
Tests the backfill operation `hash_sources(net, repos, force=False)`. This is used to populate `source_hash` on nodes that were added without one. Tests cover:
- **Backfill empty** — fills in hash, reports `was_empty: True`.
- **Skips existing** — doesn't overwrite a node that already has a hash.
- **Force mode** — `force=True` rehashes even nodes with existing hashes.
- **Missing file skipped** — no crash, no result entry.
- **No source field** — nodes without `source` are silently skipped.
- **Batch operation** — handles multiple nodes, skips missing files, reports only successes.

## Patterns

- **`tmp_path` fixture**: Every test that touches the filesystem uses pytest's `tmp_path` to create isolated, auto-cleaned directories. No test leaves artifacts.
- **Repos dictionary**: Source paths are decoupled from absolute filesystem locations via a `repos: dict[str, Path]` mapping. This lets tests point `"myrepo"` or `"r"` at `tmp_path` without touching real directories. The same indirection is used in production to map logical repository names to checkout paths.
- **Hash-then-mutate-then-check**: The stale detection tests follow a consistent pattern — write a file, record its hash, mutate the file, then assert staleness is detected.
- **Return-value contracts over exceptions**: Both `check_stale` and `hash_sources` return lists of dicts rather than raising. Empty list means "nothing to report."

## Dependencies

**Imports:**
- `hashlib` — manual hash computation for expected values
- `pathlib.Path` — via `tmp_path`
- `reasons_lib.network.Network` — the belief graph; used to create nodes with source metadata
- `reasons_lib.check_stale` — the four functions under test

**Imported by:** Nothing. This is a leaf test module.

## Flow

Each test follows the same shape:
1. Create temp files with known content.
2. Build a `Network` and add nodes with `source="repo/file"` and optionally `source_hash=<sha256>`.
3. Optionally mutate the files on disk.
4. Call `check_stale` or `hash_sources` with a repos mapping.
5. Assert the returned list of dicts matches expectations, and (for `hash_sources`) that node state was mutated.

## Invariants

- `check_stale` only examines nodes that are **IN**, have a non-empty `source`, and have a non-empty `source_hash`. All three conditions must hold.
- `check_stale` never mutates the network — it's a read-only scan.
- `hash_sources` **does** mutate nodes — it writes `source_hash` onto nodes in-place.
- `hash_sources` without `force=True` is idempotent on nodes that already have hashes.
- Source paths are always in `"repo/relative"` format; `resolve_source_path` splits on the first `/` to find the repo key.

## Error Handling

No exceptions are expected or tested. All "error" conditions (missing file, empty source, missing hash) are handled by returning `None` or empty lists. The design favors silent skipping over raising — appropriate for a batch scan operation where partial results are useful.

---

## Topics to Explore

- [file] `reasons_lib/check_stale.py` — The implementation of `check_stale`, `hash_sources`, and `resolve_source_path`; see how the repos dict is parsed and how source paths are split
- [function] `reasons_lib/network.py:add_node` — How `source` and `source_hash` are stored on Node objects; the kwargs contract that these tests depend on
- [file] `tests/test_check_stale_issue25.py` — A companion test file likely covering a specific bug fix related to staleness detection
- [general] `staleness-vs-retraction` — How staleness detection feeds into the TMS retraction mechanism; `check_stale` reports but doesn't retract — something upstream must act on results
- [function] `reasons_lib/cli.py:check-stale` — The CLI entrypoint that calls `check_stale` in production, including how the repos mapping is constructed from arguments or config

## Beliefs

- `check-stale-skips-out-nodes` — `check_stale` only inspects nodes with truth value IN; retracted (OUT) nodes are excluded from staleness checks
- `check-stale-is-read-only` — `check_stale` returns a list of dicts and never mutates the Network; `hash_sources` is the mutating counterpart
- `source-path-format` — Source references use `"reponame/relative/path"` format, resolved via a `repos` dict mapping repo names to filesystem `Path` objects
- `hash-sources-idempotent-without-force` — `hash_sources` with default `force=False` skips nodes that already have a `source_hash`, making repeated calls safe
- `stale-result-reasons` — Stale results use two distinct reason codes: `"content_changed"` when the file exists but differs, and `"source_deleted"` when the file is gone

