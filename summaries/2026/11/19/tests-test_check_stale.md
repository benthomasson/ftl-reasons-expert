---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_check_stale.py

# Explanation: `tests/test_check_stale.py`

## Purpose

This is a pytest unit-test module for `reasons.check_stale`. It pins down the behavior of stale-source detection and source-hash management:

- hashing files with SHA-256
- resolving a node’s `source` string to an actual filesystem path
- detecting when source content has changed or disappeared
- handling truncated/prefixed hashes and optional hash upgrades
- backfilling missing source hashes

The tests target the `reasons` package, not the parallel `reasons_lib` package. No production module imports this test file; pytest discovers it via its `test_*.py` filename.

## Key Components

### Imports

- `hashlib` — used to compute expected SHA-256 digests.
- `Path` — imported but not used directly in this file.
- `pytest` — imported but not used directly; `tmp_path` is injected by pytest.
- `Network` — the in-memory belief network used to create test nodes.
- `check_stale`, `hash_file`, `hash_sources`, `resolve_source_path` — functions under test.

### `TestHashFile`

Tests `hash_file(path)`.

Contracts enforced:

- Returns `hashlib.sha256(file_bytes).hexdigest()`.
- Hash changes when file content changes.

### `TestResolveSourcePath`

Tests `resolve_source_path(source, repos=..., db_dir=..., agent=...)`.

Observed contracts:

- Empty `source` returns `None`.
- Missing resolved file returns `None`.
- `db_dir` can resolve a source path relative to the database directory.
- `db_dir` takes precedence over split repo mapping when both contain the file.
- Split repo mapping resolves `"repo_key/relative/path"` to `repos["repo_key"] / "relative/path"`.
- Agent-specific repo resolution uses `repos[agent]` and can take precedence over split repo mapping.
- If a file is not found under `db_dir`, resolution falls back to split repos.
- `repos` values may be `Path` objects in direct tests and `str` values in `Network.repos`.

Important precedence evidence:

- `test_db_dir_takes_precedence`: `db_dir/entries/topic.md` wins over `repos["entries"]/topic.md`.
- `test_agent_repo_takes_precedence_over_split`: `repos["code"]/entries/topic.md` wins over `repos["entries"]/topic.md`.
- `test_falls_back_to_repo_when_not_in_db_dir`: missing file under `db_dir` falls back to split repo.

### `TestCheckStale`

Tests `check_stale(net, repos=..., db_dir=..., upgrade_hashes=...)`.

The tests unpack return values in two forms:

```python
results, *_ = check_stale(...)
results, upgraded, *_ = check_stale(...)
```

So the function returns a tuple whose first element is a list of stale-result dicts and whose second element is an upgraded-hash count. Additional elements may exist but are not tested.

Result dict keys exercised by tests:

- `node_id`
- `old_hash`
- `new_hash`
- `reason`
- `source_path`

Exact `reason` values asserted:

- `"content_changed"`
- `"source_deleted"`
- `"truncated_hash"`

Behavior enforced:

- Fresh node via `db_dir` produces no results.
- Fresh node via split repo produces no results.
- Changed source content produces `content_changed` with old/new hashes.
- Retracted/out nodes are skipped.
- Nodes with empty `source_hash` are skipped.
- Missing source files produce `source_deleted` with `new_hash is None` and `source_path is None`.
- Multiple stale nodes can be reported; only count is asserted, not order.
- Agent-imported nodes resolve through `net.repos[agent]` using `metadata={"agent": "code"}`.
- Agent-imported nodes can also be detected as stale after content change.

A subtle invariant from `test_skips_nodes_without_hash`: a node with no hash is skipped before missing-source reporting. The source file does not exist in that test, but no `source_deleted` result is expected because the node has no hash.

### `TestPrefixHashUpgrade`

Tests truncated/prefixed hash handling in `check_stale`.

Contracts enforced:

- If stored hash is a prefix of the full SHA-256 digest:
  - with `upgrade_hashes=True`: no stale result, `upgraded == 1`, and `net.nodes[id].source_hash` is replaced with the full hash.
  - without `upgrade_hashes`: one result with `reason == "truncated_hash"`, `upgraded == 0`, and the stored hash remains unchanged.
- If content genuinely changed, even with `upgrade_hashes=True`, it reports `content_changed` and does not upgrade.
- Full hash match produces no result, `upgraded == 0`, and leaves the hash unchanged.

This means prefix detection must compare the stored hash against the beginning of the actual full hash, rather than treating any mismatch as stale.

### `TestHashSources`

Tests `hash_sources(net, repos=..., db_dir=..., force=...)`.

Contracts enforced:

- Backfills empty `source_hash` values.
- Can resolve sources via `db_dir`.
- Returns a list of result dicts with at least:
  - `node_id`
  - `was_empty`
- `was_empty` is `True` when the previous hash was empty.
- Skips nodes with an existing non-empty hash unless `force=True`.
- With `force=True`, rehashes existing hashes and sets `was_empty` to `False`.
- Skips missing source files silently; no result is emitted.
- Skips nodes without a `source`.
- Updates `net.nodes[id].source_hash` as a side effect.
- Multiple nodes can be hashed; only membership/count is asserted, not order.

## Patterns

- **Class-based grouping by function under test**: `TestHashFile`, `TestResolveSourcePath`, etc.
- **Arrange–Act–Assert**: each test creates files/nodes, invokes one function, then asserts return values and side effects.
- **Filesystem isolation via `tmp_path`**: real files are written and hashed; no mocking of the filesystem.
- **Direct expected-hash computation**: tests compute expected SHA-256 with `hashlib.sha256(...).hexdigest()`.
- **Side-effect assertions**: tests inspect `net.nodes["a"].source_hash` to verify mutations.
- **Tolerant tuple unpacking**: `results, *_` and `results, upgraded, *_` allow implementation to return extra values without breaking tests.
- **No ordering assumptions**: multi-result tests assert length and membership, not exact order.
- **Internal `Network` coupling**: tests directly access `net.repos`, `net.nodes`, and call `net.retract(...)`.

## Dependencies

This test file depends on:

- Python standard library: `hashlib`, `pathlib.Path`
- pytest: `tmp_path`
- `reasons.network.Network`
- `reasons.check_stale.check_stale`
- `reasons.check_stale.hash_file`
- `reasons.check_stale.hash_sources`
- `reasons.check_stale.resolve_source_path`

It indirectly requires `Network` to support:

- `add_node(id, text, source=..., source_hash=..., metadata=...)`
- `retract(id)`
- `net.repos` as a mutable mapping
- `net.nodes` exposing node objects with mutable `source_hash`

No production code imports this test module. It is consumed by pytest/CI.

## Flow

A typical stale-detection test follows this flow:

1. Write a source file under `tmp_path`.
2. Compute its SHA-256 hash.
3. Create a `Network`.
4. Add a node with `source="myrepo/source.md"` and `source_hash=old_hash`.
5. Possibly mutate the source file or retract the node.
6. Call `check_stale(net, repos={"myrepo": tmp_path})`.
7. Assert the returned stale-result list, reason strings, hashes, and node state.

A typical hash-backfill test follows:

1. Write a source file.
2. Add a node with a `source` but no `source_hash`.
3. Call `hash_sources(net, repos=...)`.
4. Assert the result records the node and `was_empty` status.
5. Assert `net.nodes[id].source_hash` is no longer empty.

Path resolution flow inferred from tests:

- Empty source → `None`.
- Try agent-specific repo when `agent` is provided and available.
- Try `db_dir / source`.
- Split `source` into repo key and relative path, then try `repos[repo_key] / relative_path`.
- Return resolved `Path` only if it exists; otherwise `None`.

## Invariants

- `hash_file` returns a lowercase SHA-256 hex digest of the file content.
- Empty `source` resolves to `None`.
- Missing resolved source file resolves to `None`.
- `check_stale` skips retracted/out nodes.
- `check_stale` skips nodes whose `source_hash` is empty.
- Missing source file for a node that has a hash produces `reason == "source_deleted"`.
- For `source_deleted`, `new_hash is None` and `source_path is None`.
- For `content_changed`, `new_hash != old_hash`.
- A stored hash that is a prefix of the actual full hash is treated as truncated, not as content change.
- Truncated-hash upgrade only occurs when `upgrade_hashes=True`.
- Successful upgrade increments the returned `upgraded` count and mutates `net.nodes[id].source_hash`.
- `hash_sources` backfills empty hashes by default.
- `hash_sources` does not overwrite existing non-empty hashes unless `force=True`.
- `hash_sources` silently skips missing source files.
- Result ordering for multiple stale/hashed nodes is not enforced by these tests.

## Error Handling

The tests expect normal return values rather than exceptions for common failure cases:

- Missing source file in `resolve_source_path` → `None`.
- Empty source in `resolve_source_path` → `None`.
- Missing source file in `check_stale` → result with `reason == "source_deleted"`, not an exception.
- Missing source file in `hash_sources` → skipped silently.
- Node without source in `hash_sources` → skipped silently.

There are no `pytest.raises` assertions in this file. Filesystem permission errors, unreadable files, and unexpected I/O failures are not covered here.

## Topics to Explore

- [file] `reasons/check_stale.py` — Implementation under test; verify exact signatures, return tuple shape, path precedence, and prefix-hash logic.
- [function] `reasons/check_stale.py:resolve_source_path` — Confirm precedence between agent repo, `db_dir`, and split repo mappings, including fallback behavior.
- [function] `reasons/check_stale.py:check_stale` — Inspect how stale-result dicts are built and when `upgrade_hashes` mutates `Network` nodes.
- [function] `reasons/network.py:Network.add_node` — Understand default `source_hash`, metadata storage, and how `net.repos` is represented.
- [file] `tests/test_check_stale_issue25.py` — Additional regression coverage for the same module; check whether it covers edge cases not present here.

## Beliefs

- `check-stale-returns-results-and-upgrade-count` — `check_stale` returns a tuple whose first element is a stale-result list and whose second element is an upgraded-hash count.
- `source-hash-is-sha256-hexdigest` — `hash_file` and stored `source_hash` values are SHA-256 hex digests of file content.
- `empty-source-hash-skips-stale-check` — `check_stale` skips nodes with an empty `source_hash`, even if their source file is missing.
- `missing-source-produces-source-deleted-result` — `check_stale` reports a node whose resolved source file is missing with `reason == "source_deleted"`, `new_hash is None`, and `source_path is None`.
- `prefix-hash-upgrade-requires-flag` — A stored hash that is a prefix of the actual SHA-256 digest is only upgraded when `upgrade_hashes=True`; otherwise it is reported as `truncated_hash`.
- `hash-sources-backfills-empty-hashes-unless-force` — `hash_sources` updates nodes with empty `source_hash` by default and only rehashes existing non-empty hashes when `force=True`.
