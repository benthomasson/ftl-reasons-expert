# File: tests/test_check_stale_issue25.py

**Date:** 2026-04-29
**Time:** 17:01

No worries — I have enough from the test file itself and the project context to give you a solid explanation.

---

## Purpose

`test_check_stale_issue25.py` is a **regression test suite for GitHub issue #25**. It validates a specific behavioral fix: `check_stale()` must report `source_deleted` results when a belief's source file no longer exists on disk, rather than silently skipping those nodes.

This matters because beliefs in the TMS are grounded in source files (e.g., markdown exploration entries). If a source file is deleted but the belief remains IN, the system has a dangling reference. Before issue #25, `check_stale()` would silently ignore this case — the fix makes it explicit by returning a result with `reason: "source_deleted"`.

## Key Components

### Helper: `_hash(content: str) -> str`
A convenience wrapper around `hashlib.sha256` for constructing expected hashes in test fixtures. Mirrors `hash_file` from the production code but operates on strings rather than file paths.

### `TestSourceDeletedResult` — Core behavior (3 tests)
Validates the happy path of the fix:
- **`test_missing_source_returns_source_deleted`**: A node referencing `r/gone.md` (which doesn't exist in `tmp_path`) produces a result with `reason="source_deleted"`, `new_hash=None`, `source_path=None`.
- **`test_source_deleted_has_all_required_keys`**: The result dict contains exactly the 6-key schema: `{node_id, old_hash, new_hash, source, source_path, reason}`.
- **`test_content_changed_has_same_keys_as_source_deleted`**: Both `source_deleted` and `content_changed` results share the same key set — consumers don't need to handle different shapes.

### `TestMixedResults` — Composition (2 tests)
Validates that deleted and changed sources coexist in the same run:
- **`test_mixed_deleted_and_changed`**: Three nodes — one changed, one deleted, one fresh. Only the first two appear in results, with correct reasons. Fresh nodes are excluded.
- **`test_multiple_nodes_same_deleted_source`**: Two nodes pointing at the same missing file both get reported independently (no dedup by file path).

### `TestSkipBehavior` — Negative cases (4 tests)
Validates that `check_stale()` correctly **skips** nodes that shouldn't produce results:
- **OUT nodes** (retracted beliefs) — even if their source is missing
- **Nodes without `source`** metadata
- **Nodes without `source_hash`** metadata
- **Nodes with empty string `source`**

### `TestEdgeCases` — Boundary conditions (6 tests)
- **Empty files**: An empty file with the correct hash is fresh (not deleted). With the wrong hash, it's `content_changed`.
- **`repos=None`**: When no repo mapping is provided, missing files still produce `source_deleted`.
- **Fresh nodes**: Confirm they produce zero results.
- **Sort order**: Results are sorted by `node_id` — deterministic output for consumers.
- **`source_path` population**: `content_changed` results populate `source_path` with the resolved filesystem path.

## Patterns

**Repo mapping convention**: Source strings use a `repo_prefix/path` format (e.g., `r/gone.md`). The `repos` dict maps prefixes to filesystem roots (`{"r": tmp_path}`). This is how `check_stale` resolves logical source references to actual file paths.

**`tmp_path` fixture**: Every test uses pytest's `tmp_path` to create an isolated filesystem. Files are created, modified, or left absent to simulate different staleness scenarios. The pattern is always: set up network → manipulate files → call `check_stale` → assert on results.

**Result dict contract testing**: Several tests explicitly verify the key set of result dicts, not just values. This guards against schema drift — if `check_stale` adds or removes a key, these tests catch it.

**Uniform result shape**: The test `test_content_changed_has_same_keys_as_source_deleted` enforces that all result types share the same dict schema. This is an intentional design choice — it means consumers can iterate results without type-checking.

## Dependencies

**Imports:**
- `reasons_lib.network.Network` — the in-memory belief graph. Used here to create nodes with `source` and `source_hash` metadata, and to retract nodes.
- `reasons_lib.check_stale.check_stale` — the function under test. Takes a `Network` and optional `repos` mapping, returns a list of staleness result dicts.
- `reasons_lib.check_stale.hash_file` — imported but not used in this file (likely kept for documentation/availability).

**Imported by:** Nothing — this is a leaf test module.

## Flow

Every test follows the same three-phase pattern:

1. **Arrange**: Create a `Network`, add nodes with `source` and `source_hash` metadata, optionally create/modify files under `tmp_path`.
2. **Act**: Call `check_stale(net, repos={"r": tmp_path})`.
3. **Assert**: Check the returned list of result dicts for expected `reason`, `node_id`, hash values, and key sets.

`check_stale` internally iterates IN nodes that have both `source` and `source_hash`, resolves the source path via the repo mapping, checks if the file exists, hashes it if present, and compares against `source_hash`. The result is a sorted list of dicts describing mismatches.

## Invariants

1. **Result schema is fixed**: Every result dict has exactly 6 keys: `{node_id, old_hash, new_hash, source, source_path, reason}`.
2. **`source_deleted` implies `new_hash=None` and `source_path=None`**: There's no file to hash or point at.
3. **`content_changed` implies `source_path` is populated**: The file exists but has different content.
4. **Only IN nodes are checked**: Retracted (OUT) nodes are skipped regardless of source state.
5. **Both `source` and `source_hash` are required**: Nodes missing either are skipped.
6. **Results are sorted by `node_id`**: Enforced by `test_results_sorted_by_node_id`.
7. **No dedup by source path**: Multiple nodes referencing the same missing file each produce their own result.

## Error Handling

There is no error handling in this test file — it's pure assertion-based testing. The interesting error-handling behavior is in `check_stale()` itself: the fix for issue #25 is specifically about *not* swallowing the "file doesn't exist" case. Before the fix, a missing file was silently ignored. After the fix, it produces an explicit `source_deleted` result. The tests validate this by ensuring the result list is non-empty when source files are absent.

## Topics to Explore

- [file] `reasons_lib/check_stale.py` — The implementation being tested; see how repo prefix resolution, file hashing, and the `source_deleted` code path work
- [function] `reasons_lib/network.py:Network.add_node` — How `source` and `source_hash` metadata are stored on nodes; the contract this test suite depends on
- [file] `tests/test_check_stale.py` — The original staleness test suite (pre-issue-25); compare coverage to understand what the regression tests add
- [general] `repo-prefix-resolution` — How the `repos` dict maps logical source prefixes like `r/` to filesystem paths; affects all staleness checking
- [diff] `issue-25-fix` — The PR that introduced `source_deleted` handling; shows what changed in `check_stale()` to produce these results

## Beliefs

- `check-stale-source-deleted-returns-none-hashes` — When a source file is missing, `check_stale` returns `new_hash=None` and `source_path=None` in the result dict
- `check-stale-result-schema-uniform` — All `check_stale` result dicts share the same 6-key schema (`node_id`, `old_hash`, `new_hash`, `source`, `source_path`, `reason`) regardless of reason type
- `check-stale-skips-out-nodes` — `check_stale` does not report staleness for retracted (OUT) nodes, even if their source files are missing
- `check-stale-results-sorted-by-node-id` — The list returned by `check_stale` is sorted ascending by `node_id`
- `check-stale-no-dedup-by-source-path` — Multiple nodes referencing the same missing source file each produce independent `source_deleted` results

