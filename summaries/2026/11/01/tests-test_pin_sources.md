---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_pin_sources.py

# `tests/test_pin_sources.py`

## Purpose

This file is the test suite for **git-aware source pinning** — a mechanism that lets beliefs (nodes in a `Network`) anchor themselves to a specific commit SHA of a source file, so that later staleness checks can distinguish between "the file actually changed in the lines I care about" versus "the file was touched but not in a way that invalidates my belief."

The responsibility it owns is verifying the contract of the `reasons.check_stale` module's git-integration layer: commit SHA resolution, file-change detection, URL rewriting, line-range pinning, diff-hunk parsing, and the `check_stale` decision logic itself.

## Key Components

| Component | Contract |
|-----------|----------|
| `git_repo` fixture | Yields a `tmp_path` containing a freshly `git init`'d repo with one committed file (`source.md` → `"initial content"`). All tests in the file depend on this. |
| `_get_head_sha(repo_path)` | Returns the 40-char SHA of `HEAD` via `git rev-parse`. |
| `_commit_change(repo_path, filename, content, message)` | Writes new content, stages, and commits in one shot. |
| `TestGetFileCommitSha` | Verifies `get_file_commit_sha` returns a 40-char SHA for tracked files, `None` for untracked files or files outside a repo, and that the SHA changes after a new commit. |
| `TestFileChangedSince` | Verifies `file_changed_since(path, sha)` returns `False` when no commits touch the file since `sha`, `True` when they do, and ignores unrelated file changes. |
| `TestPinSourceUrl` | Verifies `pin_source_url(url, sha)` rewrites GitHub `/blob/<branch>/` to `/blob/<sha>/`, passes through non-GitHub URLs, and handles slashed branch names via a `source_path` hint. |
| `TestPinSources` | Verifies `pin_sources(net, db_dir=...)` bulk-pins all non-OUT nodes with a `source` field, backfills `source_hash`, skips already-pinned nodes (unless `force=True`), skips OUT nodes / nodes without sources / nonexistent sources, and optionally rewrites `source_url`. |
| `TestPinUpdate` | Verifies `pin_update(net, [ids], db_dir=...)` re-pins specific nodes, refreshes `source_hash`, and returns structured error dicts (`"not found"`, `"no source"`, `"source not found"`). |
| `TestCheckStaleGitAware` | Verifies the `check_stale(net, db_dir=..., git_aware=True)` decision: fresh when no commits since pin, stale on content change, auto-bumps SHA when file content reverted, falls back to hash-only when no `pinned_sha`, and respects `git_aware=False`. |
| `TestParseDiffHunks` | Verifies `parse_diff_hunks(diff_text)` extracts `(start, end)` line ranges from `@@` headers, handles the no-count shorthand (`-42` → line 42), excludes zero-count hunks, and returns `[]` for empty/no-hunk input. |
| `TestFileLinesChangedSince` | Verifies `file_lines_changed_since(path, sha, start, end)` detects changes only within the pinned line range, ignores out-of-range edits, works on uncommitted changes, and fails open (returns `True`) on an invalid SHA. |
| `TestPinLines` | Verifies `pin_lines(net, node_id, start, end, db_dir=...)` stores `"start-end"` in `metadata["pinned_lines"]`, auto-pins SHA if missing, preserves existing SHA, validates `start >= 1` and `start <= end`, and raises `KeyError`/`ValueError` on bad input. |
| `TestCheckStaleLineRange` | Verifies `check_stale` with `pinned_lines` in metadata: fresh when change is outside range, stale when inside, falls back to whole-file check when no `pinned_lines`, and preserves `pinned_lines` through auto-bump while refreshing `source_hash`. |

## Patterns

1. **Fixture-based isolation**: The `git_repo` fixture creates a disposable repo per test (via pytest's `tmp_path`), ensuring no cross-test contamination. Each test that needs a multi-line file calls a local helper (`_multiline_file`, `_setup_multiline`) to reset state.

2. **Subprocess-as-test-harness**: Rather than mocking `git`, the tests drive real `git` via `subprocess.run(..., capture_output=True, check=True)`. This exercises the actual shell commands the production code likely invokes, catching quoting/path issues that a mock would hide.

3. **Result-dict assertion style**: Functions like `pin_sources`, `pin_update`, and `check_stale` return lists of dicts (or tuples). Tests assert on dict keys (`"node_id"`, `"pinned_sha"`, `"was_empty"`, `"reason"`, `"error"`) rather than on exceptions — a "errors-as-data" pattern.

4. **Parametric boundary testing**: `TestPinLines.test_validates_line_range` and `TestParseDiffHunks` cover edge cases (line 0, reversed range, single-line hunk with no count, zero-count hunk) to pin down the exact boundary semantics.

5. **Fail-open semantics tested explicitly**: `test_fails_open_on_bad_sha` asserts that an invalid SHA causes `file_lines_changed_since` to return `True` (i.e., "assume stale"), not raise. This is a safety invariant for a staleness checker.

## Dependencies

**Imports (this file depends on):**
- `pytest` — test runner, fixtures
- `subprocess`, `hashlib`, `pathlib` — git interactions and hashing
- `reasons.network.Network` — the belief graph data structure
- `reasons.check_stale` — the module under test (`check_stale`, `get_file_commit_sha`, `file_changed_since`, `file_lines_changed_since`, `hash_file`, `parse_diff_hunks`, `pin_lines`, `pin_source_url`, `pin_sources`, `pin_update`)

**Depended upon by (imported_by):**
- This is a test file; nothing imports it at runtime. It is discovered by `pytest` via the `tests/` package (see `tests/__init__.py`).
- CI workflow in `.github/workflows/` invokes the test suite.

## Flow

The execution flow for a representative test (`TestCheckStaleGitAware.test_stale_when_content_changed`):

1. **Fixture setup** (`git_repo`): `git init` → write `source.md` → `git add` → `git commit`.
2. **Test body**:
   a. `_get_head_sha` → `git rev-parse HEAD` → captures the initial SHA.
   b. `hash_file` → computes content hash of `source.md`.
   c. Build a `Network`, add a node with `source="source.md"`, set `metadata["pinned_sha"]` to the captured SHA.
   d. `_commit_change` → writes `"changed content"` → `git add` → `git commit`.
   e. Call `check_stale(net, db_dir=git_repo, git_aware=True)`.
3. **Inside `check_stale`** (production code, implied by assertions):
   - Reads `pinned_sha` from node metadata.
   - Runs `git log --oneline <sha>..HEAD -- source.md` (or equivalent) to detect commits since the pin.
   - Finds a commit → marks node stale with `reason="content_changed"`.
4. **Assertion**: `results[0]["reason"] == "content_changed"`.

For `TestPinSourceUrl`, the flow is purely functional — no git involved:
- Input: URL + SHA → output: rewritten URL (or unchanged if non-GitHub).

## Invariants

- **SHA is always 40 hex chars**: Every test that inspects a SHA asserts `len(sha) == 40`. The system never stores abbreviated SHAs.
- **OUT nodes are immutable w.r.t. pinning**: `pin_sources` skips any node with `truth_value == "OUT"`. Once a belief is retracted, it is never re-pinned.
- **Idempotency without `force`**: If `metadata["pinned_sha"]` already exists, `pin_sources` returns 0 results unless `force=True`.
- **Line ranges are 1-indexed and inclusive**: `pin_lines(net, id, 89, 93)` stores `"89-93"`. Validation rejects `start < 1` and `start > end`.
- **Fail-open on git errors**: `file_lines_changed_since` returns `True` (stale) when the SHA is invalid — never raises, never returns `False` on uncertainty.
- **`pinned_lines` survives auto-bump**: When `check_stale` auto-bumps the SHA (file content reverted), the `pinned_lines` metadata is preserved.

## Error Handling

The code under test uses two distinct error strategies:

1. **Errors-as-dicts** (no exceptions): `pin_update` returns `[{"error": "not found"}]`, `[{"error": "no source"}]`, `[{"error": "source not found"}]`. Callers inspect the dict rather than catching exceptions. Tests assert on the `"error"` key.

2. **Exceptions for programmer errors**: `pin_lines` raises `ValueError` for invalid line ranges (0 or reversed) and `KeyError` for nonexistent node IDs. Tests use `pytest.raises(...)` to confirm these are loud failures, not silent skips.

3. **Silent skip (not an error)**: `pin_sources` simply omits nodes that lack a `source` field, have `truth_value="OUT"`, or reference nonexistent files. No error is recorded — the node is just not pinned.

4. **Fail-open on git**: `file_lines_changed_since` with a bogus SHA returns `True` (assume stale) rather than raising. This is a deliberate safety choice — better to flag a belief for re-verification than to silently mark it fresh.

## Topics to Explore

- [file] `reasons/check_stale.py` — The production module under test; understanding its `check_stale` decision tree and how it shells out to `git` is essential for interpreting these assertions.
- [file] `reasons/network.py` — Defines `Network` and its `Node` class (with `metadata`, `truth_value`, `source_hash`); needed to understand the data model the pinning functions mutate.
- [function] `reasons/check_stale.py:parse_diff_hunks` — The regex/logic that extracts line ranges from `git diff` output; the tests here pin its exact behavior, but the implementation details matter for edge cases.
- [general] `git-awareness-strategy` — How the project decides between hash-based staleness (content hash) and git-based staleness (commit range + line range); the `git_aware` flag and fallback logic are a key architectural choice.
- [file] `tests/conftest.py` — May define shared fixtures or `Network` factory helpers that these tests rely on implicitly.

## Beliefs

- `pin-sources-skips-out-nodes` — `pin_sources` unconditionally excludes nodes with `truth_value == "OUT"`; no parameter overrides this.
- `file-lines-changed-since-fails-open` — `file_lines_changed_since` returns `True` (stale) when given an invalid SHA, never `False` or an exception, to avoid silently marking beliefs as fresh.
- `pinned-lines-survive-auto-bump` — When `check_stale` auto-bumps `pinned_sha` (content reverted to match hash), the `metadata["pinned_lines"]` field is preserved unchanged.
- `pin-source-url-rewrites-github-only` — `pin_source_url` only rewrites URLs matching the GitHub `/blob/<ref>/` pattern; all other URLs are returned verbatim regardless of the SHA argument.
- `parse-diff-hunks-excludes-zero-count` — A hunk header like `@@ -89,0 +90,3 @@` (zero old-side count) is excluded from the returned list, treating pure insertions as "no lines changed in the old range."
