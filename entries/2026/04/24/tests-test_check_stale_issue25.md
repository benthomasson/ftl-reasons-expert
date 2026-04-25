# File: tests/test_check_stale_issue25.py

**Date:** 2026-04-24
**Time:** 16:59

## Explanation: `tests/test_check_stale_issue25.py`

### 1. Purpose

This is a regression test file for **issue #25**: before the fix, `check_stale()` silently skipped nodes whose source files had been deleted from disk. After the fix, it returns a structured `source_deleted` result for each missing file. The file's sole responsibility is ensuring that behavior works correctly and doesn't regress.

It tests the `check_stale()` function from `reasons_lib/check_stale.py`, specifically the branch at line 69–77 where `resolve_source_path()` returns `None`.

### 2. Key Components

**`_hash(content: str) -> str`** — Test helper that mirrors the production `hash_file()` but works on string content instead of file paths. Used to set up nodes with known-correct hashes so tests can distinguish "content changed" from "source deleted."

**`TestSourceDeletedResult`** (3 tests) — Verifies the core contract of the fix:
- A node pointing at a nonexistent file gets `reason: "source_deleted"` with `new_hash: None` and `source_path: None`
- The result dict has exactly the 6 required keys (`node_id`, `old_hash`, `new_hash`, `source`, `source_path`, `reason`)
- `source_deleted` results have the same key set as `content_changed` results — this is an API uniformity guarantee

**`TestMixedResults`** (2 tests) — Validates that a single `check_stale()` call can return both `content_changed` and `source_deleted` results simultaneously, and that multiple nodes referencing the same deleted file each produce their own result.

**`TestSkipBehavior`** (4 tests) — Guards the filter logic at `check_stale.py:63–66`. Nodes are excluded from staleness checking when they are:
- OUT (retracted)
- Missing a `source` field
- Missing a `source_hash` field
- Have an empty string as `source`

**`TestEdgeCases`** (5 tests) — From reviewer notes (likely code review on the PR). Covers:
- An empty file on disk is not treated as deleted
- An empty file with a wrong hash reports `content_changed`, not `source_deleted`
- `repos=None` falls back to `~/git/` resolution, which also produces `source_deleted` for nonexistent files
- Fresh (hash-matching) nodes produce no results
- Results are sorted by `node_id` (matches `sorted(network.nodes.items())` at line 62)

### 3. Patterns

- **`tmp_path` fixture**: Every test uses pytest's `tmp_path` to create isolated filesystem state. The `repos={"r": tmp_path}` mapping makes the source string `"r/gone.md"` resolve to `tmp_path / "gone.md"`, avoiding any real filesystem dependencies.

- **Arrange-Act-Assert**: Each test follows strict AAA — build a `Network`, optionally create/modify files, call `check_stale()`, assert on the result list.

- **Test class grouping by concern**: Rather than one flat list of tests, they're organized by what aspect of the contract they verify (core behavior, mixed scenarios, skip rules, edge cases). This matches a "specification by example" style.

- **Schema assertions**: Multiple tests assert on the exact key set of result dicts (`set(results[0].keys()) == required_keys`), treating the return shape as a contract.

### 4. Dependencies

**Imports:**
- `reasons_lib.network.Network` — the in-memory belief graph; used here only as a container to set up nodes with `source` and `source_hash` attributes
- `reasons_lib.check_stale.check_stale` — the function under test
- `reasons_lib.check_stale.hash_file` — imported but never used in the test body (likely a leftover from development or kept for potential future use)

**Imported by:** Nothing — this is a leaf test file.

### 5. Flow

Every test follows the same pattern:

1. Create a `Network` and add nodes with `source="r/<filename>"` and a `source_hash`
2. Optionally create or modify files under `tmp_path` to control what the filesystem looks like
3. Call `check_stale(net, repos={"r": tmp_path})` — this iterates all IN nodes with source+hash, resolves each source path, and compares hashes or detects missing files
4. Assert on the returned list of dicts

The `repos` mapping is the key indirection: it lets tests substitute `tmp_path` for what would normally be `~/git/r/` in production, making everything hermetic.

### 6. Invariants

The tests collectively enforce these rules:

- **Every IN node with a `source` and `source_hash` must be checked** — nothing is silently skipped
- **Missing file → `source_deleted`**, never an exception or empty result
- **`source_deleted` and `content_changed` share identical key sets** — consumers can handle both uniformly
- **OUT nodes, no-source nodes, no-hash nodes, and empty-source nodes are excluded** — four distinct skip conditions
- **Results are sorted by `node_id`** — deterministic ordering for consumers
- **Empty files are real files** — zero-length content is not confused with deletion

### 7. Error Handling

There is no error handling in these tests, which is the point: `check_stale()` must never raise on missing files. The pre-fix behavior likely raised `FileNotFoundError` or silently skipped; the fix converts that into a structured result. The tests confirm that the function handles all degenerate inputs (missing files, empty strings, `None` repos) without exceptions.

The unused `hash_file` import is the only code smell — it's harmless but suggests an earlier draft of the tests that directly tested `hash_file` on missing paths.

---

## Topics to Explore

- [file] `reasons_lib/check_stale.py` — The implementation under test; see `resolve_source_path()` for how `repos` mapping works and the fallback to `~/git/`
- [file] `tests/test_check_stale.py` — The original staleness tests; compare to understand what issue #25 added beyond the baseline coverage
- [function] `reasons_lib/check_stale.py:resolve_source_path` — The path resolution logic that returns `None` for missing files, which is the trigger for `source_deleted` results
- [function] `reasons_lib/network.py:add_node` — How `source` and `source_hash` are stored on nodes; the fields these tests depend on
- [general] `staleness-and-retraction-integration` — How stale detection feeds into the CLI (`check-stale` command) and whether `source_deleted` automatically retracts nodes or just reports

---

## Beliefs

- `check-stale-never-raises-on-missing-files` — `check_stale()` returns a `source_deleted` dict for nodes whose source files don't exist; it never raises `FileNotFoundError`
- `result-schema-uniformity` — Both `content_changed` and `source_deleted` results have exactly the same six keys: `node_id`, `old_hash`, `new_hash`, `source`, `source_path`, `reason`
- `out-nodes-excluded-from-staleness` — `check_stale()` only examines nodes with `truth_value == "IN"`; retracted (OUT) nodes are skipped even if their source is missing
- `results-sorted-by-node-id` — `check_stale()` returns results sorted lexicographically by `node_id`, enforced by iterating `sorted(network.nodes.items())`
- `hash-file-import-unused` — `hash_file` is imported in this test file but never called; all hashing goes through the local `_hash()` helper

