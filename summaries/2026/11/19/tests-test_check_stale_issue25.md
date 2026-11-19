---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_check_stale_issue25.py

# `tests/test_check_stale_issue25.py`

## Purpose

This is a regression test module for issue #25. Its job is to pin down the behavior of `reasons.check_stale.check_stale()` when a node references a source file that no longer exists on disk.

The core bug/contract it protects is:

- `check_stale()` must report missing source files as `source_deleted`
- it must not silently skip such nodes
- deleted-source results must have the same schema as other stale results
- other legitimate skip rules must still apply

It is not the general `check_stale` test suite; that lives in `tests/test_check_stale.py`. This file focuses specifically on deleted-source handling, result schema, mixed results, skip behavior, and reviewer edge cases.

There are 15 test methods grouped into four test classes.

---

## Key Components

### `_hash(content: str) -> str`

Small test helper that computes a SHA-256 hex digest:

```python
hashlib.sha256(content.encode()).hexdigest()
```

It is used to build expected hashes for source files without relying on the production `hash_file` helper.

Note: `hash_file` is imported from `reasons.check_stale` but is not used in this file.

---

### `TestSourceDeletedResult`

Validates the core deleted-source result contract.

Tests:

- `test_missing_source_returns_source_deleted`
  - Adds node `"a"` with `source="r/gone.md"` and `source_hash="abc123"`.
  - Calls `check_stale(net, repos={"r": tmp_path})`.
  - Expects exactly one result with:
    - `node_id == "a"`
    - `reason == "source_deleted"`
    - `new_hash is None`
    - `source_path is None`
    - `old_hash == "abc123"`
    - `source == "r/gone.md"`

- `test_source_deleted_has_all_required_keys`
  - Asserts the result dict has exactly these keys:
    - `node_id`
    - `old_hash`
    - `new_hash`
    - `source`
    - `source_path`
    - `reason`
  - Because it uses set equality, extra keys would fail.

- `test_content_changed_has_same_keys_as_source_deleted`
  - Creates one changed source and one deleted source.
  - Asserts both `content_changed` and `source_deleted` results have the same key set.

---

### `TestMixedResults`

Validates behavior when deleted and changed sources appear in the same run.

Tests:

- `test_mixed_deleted_and_changed`
  - Creates:
    - `changing.md`, initially hashed as old content
    - `stable.md`, hashed as stable content
    - node `"changed"` pointing at `changing.md`
    - node `"deleted"` pointing at missing `gone.md`
    - node `"fresh"` pointing at `stable.md`
  - Rewrites `changing.md` after hashing.
  - Asserts:
    - `changed` => `content_changed`
    - `deleted` => `source_deleted`
    - `fresh` is absent from results

- `test_multiple_nodes_same_deleted_source`
  - Adds two nodes `"a"` and `"b"` referencing the same missing source.
  - Expects two separate `source_deleted` results, one per node.
  - Confirms missing-source reporting is per node, not deduplicated by source path.

---

### `TestSkipBehavior`

Validates which nodes should still be ignored by `check_stale()`.

Tests:

- `test_skips_out_nodes_with_missing_source`
  - Adds node, then calls `net.retract("a")`.
  - Even though its source is missing, result list is empty.
  - Confirms retracted/out nodes are skipped before source checks.

- `test_skips_nodes_without_source`
  - Node has no `source`.
  - Result list empty.

- `test_skips_nodes_without_hash`
  - Node has `source` but no `source_hash`.
  - Result list empty.

- `test_skips_nodes_with_empty_source_string`
  - Node has `source=""` and a hash.
  - Result list empty.
  - Empty source is treated as “no source”, not as a missing file.

---

### `TestEdgeCases`

Covers reviewer-noted edge cases.

Tests:

- `test_empty_file_is_not_deleted`
  - Creates an empty file and sets node hash to `_hash("")`.
  - Expects no result.
  - Confirms an existing empty file is fresh, not deleted.

- `test_empty_file_with_wrong_hash_is_content_changed`
  - Empty file exists, but stored hash is wrong.
  - Expects one `content_changed` result with `new_hash == _hash("")`.

- `test_no_repos_mapping_with_missing_file`
  - Calls `check_stale(net, repos=None)` with source `"nonexistent-repo/file.md"`.
  - Expects one `source_deleted` result.
  - Confirms `repos=None` is valid and does not suppress deleted-source reporting.

- `test_fresh_node_not_in_results`
  - File exists and hash matches.
  - Expects empty result list.

- `test_results_sorted_by_node_id`
  - Adds `"z-node"`, `"a-node"`, `"m-node"`, all with missing sources.
  - Asserts result IDs are sorted ascending.

- `test_content_changed_populates_source_path`
  - Creates file, stores old hash, rewrites file.
  - Expects one `content_changed` result.
  - Asserts `source_path == str(f)`.
  - This contrasts with `source_deleted`, where `source_path is None`.

---

## Patterns

- **Pytest class grouping**: tests are grouped by behavior into classes with descriptive names.
- **`tmp_path` fixture**: every filesystem test gets an isolated temporary directory.
- **Arrange-Act-Assert**: each test builds a `Network`, mutates files, calls `check_stale`, then asserts on results.
- **Real filesystem, no mocks**: source files are actually created, rewritten, or omitted.
- **Tuple destructuring**:
  ```python
  results, *_ = check_stale(...)
  ```
  This encodes the contract that `check_stale()` returns a tuple-like value whose first element is the result list. Additional return values are ignored by these tests.
- **Exact schema assertions**: `set(results[0].keys()) == required_keys` enforces no extra result keys.
- **Reason-based selection**: `next(r for r in results if r["reason"] == ...)` picks records by reason.
- **Dictionary/set comprehensions for assertions**: used to map node IDs to reasons and compare node ID sets.
- **Issue-specific regression file**: the filename and module docstring explicitly identify issue #25.

---

## Dependencies

### Imports used

- `hashlib` — computes expected SHA-256 hashes in `_hash`.
- `reasons.network.Network` — in-memory graph setup for nodes and retraction.
- `reasons.check_stale.check_stale` — system under test.

### Imports present but unused

- `pathlib.Path` — imported but not referenced directly.
- `pytest` — imported but not referenced directly; `tmp_path` is injected by pytest without needing `import pytest`.
- `reasons.check_stale.hash_file` — imported but not used.

### What depends on this file

No production code imports this test module. It is discovered by pytest and used by CI/developers as a regression signal.

### Package targeting

This file imports from `reasons`, not `reasons_lib`. The repository reflects the same modules under both `reasons/` and `reasons_lib/`. Changes to `reasons_lib` alone would not be exercised by this test.

---

## Flow

A typical test follows this sequence:

1. Create an in-memory `Network`.
2. Add one or more nodes with:
   - node ID
   - belief text
   - optional `source`
   - optional `source_hash`
3. Create or mutate files under `tmp_path`.
4. Call:
   ```python
   check_stale(net, repos={"r": tmp_path})
   ```
5. Destructure the first return value:
   ```python
   results, *_ = check_stale(...)
   ```
6. Assert on:
   - result count
   - `node_id`
   - `reason`
   - `old_hash`
   - `new_hash`
   - `source`
   - `source_path`
   - ordering
   - exact key set

The `repos` mapping is keyed by repo prefix. In most tests, source strings use prefix `"r"`, and `repos={"r": tmp_path}` maps `"r/..."` to files under `tmp_path`.

For skipped cases, the expected output is simply:

```python
assert results == []
```

---

## Invariants

These tests enforce the following rules for `check_stale()`:

- A node with a non-empty `source` and non-empty `source_hash`, whose source file is missing, must produce a `source_deleted` result.
- `source_deleted` results must have:
  - `new_hash is None`
  - `source_path is None`
  - `old_hash` preserved from the node
  - `source` preserved from the node
- `content_changed` results must include:
  - computed `new_hash`
  - populated `source_path` as a string
- All result records must contain exactly:
  - `node_id`
  - `old_hash`
  - `new_hash`
  - `source`
  - `source_path`
  - `reason`
- Retracted/out nodes must be skipped.
- Nodes without `source` must be skipped.
- Nodes without `source_hash` must be skipped.
- Nodes with `source == ""` must be skipped.
- An existing empty file with a matching hash is not stale.
- An existing empty file with a wrong hash is `content_changed`.
- `repos=None` is allowed.
- Multiple nodes referencing the same missing source must each produce their own result.
- Results must be sorted ascending by `node_id` in the tested missing-source scenario.
- A fresh node whose file exists and hash matches must not appear in results.

---

## Error Handling

This test file treats missing source files as normal data, not exceptions.

- There are no `pytest.raises` blocks.
- If `check_stale()` raises on a missing source file, these tests will fail.
- Missing files are expected to surface as:
  ```python
  {"reason": "source_deleted", ...}
  ```
- `repos=None` is not treated as an error.
- Retracted nodes and nodes lacking source/hash information are silently omitted.
- Permission errors, directory paths, non-UTF8 files, and other filesystem failures are not covered here.

---

## Topics to Explore

- [file] `reasons/check_stale.py` — Implements `check_stale()` and `hash_file()`; inspect how source path resolution and result dict construction work.
- [function] `reasons/check_stale.py:check_stale` — Verify the return shape, ordering, skip rules, and missing-source behavior against this test contract.
- [file] `tests/test_check_stale.py` — General `check_stale` coverage; compare it with the issue-specific edge cases in this file.
- [function] `reasons/network.py:Network.add_node` — Understand the `source`, `source_hash`, and retraction semantics used to set up these tests.
- [general] `reasons-vs-reasons_lib` — The repository mirrors modules under both packages; confirm which package CI tests and whether both need changes.

---

## Beliefs

- `missing-source-produces-source-deleted` — A tracked node whose source file is absent produces a `check_stale` result with reason `source_deleted`.
- `source-deleted-result-nulls-new-hash-and-source-path` — `source_deleted` results set `new_hash` and `source_path` to `None` while preserving `old_hash` and `source`.
- `check-stale-result-schema-is-exact` — Every result dict must contain exactly `node_id`, `old_hash`, `new_hash`, `source`, `source_path`, and `reason`.
- `check-stale-skips-ineligible-nodes` — Nodes that are retracted, lack `source`, lack `source_hash`, or have `source == ""` are omitted from results.
- `check-stale-results-sorted-by-node-id` — Results are returned in ascending `node_id` order in the edge-case test.
