---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_pin_sources.py

# `tests/test_pin_sources.py` — Test Suite for Git-Aware Source Pinning

## Purpose

This is a **pytest test module** that exercises the git-aware source-pinning subsystem implemented in `reasons/check_stale.py` (and `reasons/network.py`). Its responsibility is to lock down the contract of ten public functions that let a `Network` belief (node) be anchored to:

- a specific **git commit SHA** (`pinned_sha`),
- a full-file **content hash** (`source_hash`),
- an optional **line range** (`pinned_lines`), and
- a **permalink URL** derived from a GitHub URL + SHA.

The suite verifies both the *functional behaviour* (SHA retrieval, stale detection, diff parsing) and, crucially, the **metadata side-effects** on `network.nodes[id].metadata` — because "pinning" is state that is persisted into the network and later consumed by `pin_update` / `check_stale`.

---

## Key Components

### Fixtures & Helpers

| Symbol | Purpose |
|---|---|
| `git_repo(tmp_path)` (fixture) | Bootstraps a throwaway git repo inside `tmp_path`, configures `user.email`/`user.name`, commits a single file `source.md` with content `"initial content"`. Returns the repo root `Path`. |
| `_get_head_sha(repo_path)` | Thin `git rev-parse HEAD` wrapper returning the 40-char SHA string. |
| `_commit_change(repo_path, filename, content, message="change")` | Writes content, `git add`s, and `git commit`s — used to simulate upstream changes. |

### Test Classes (grouped by SUT function)

| Class | Function(s) exercised | Contract verified |
|---|---|---|
| `TestGetFileCommitSha` | `get_file_commit_sha(path)` | Returns the 40-char SHA of the last commit touching `path`; **`None`** for untracked files, non-git directories, and (implicitly) unknown paths. |
| `TestFileChangedSince` | `file_changed_since(path, sha)` | True iff `path` has commits after `sha`; ignores unrelated files. |
| `TestPinSourceUrl` | `pin_source_url(url, sha, source_path=None)` | Rewrites `.../blob/<branch>/<path>` → `.../blob/<sha>/<path>` for GitHub URLs. Non-GitHub URLs pass through unchanged. `source_path` disambiguates branches that contain `/`. |
| `TestPinSources` | `pin_sources(net, db_dir, force=False, pin_urls=False)` | Bulk operation returning a list of `{node_id, pinned_sha, was_empty}` dicts. Skips pinned nodes (unless `force`), `OUT` truth values, missing sources, and nodes without a source. Backfills `source_hash`. Mutates `node.metadata["pinned_sha"]` and `"verified_at"`; optionally rewrites `node.source_url`. |
| `TestPinUpdate` | `pin_update(net, node_ids, db_dir)` | Per-node re-pin. **Never raises** — returns `{error: "not found" / "no source" / "source not found"}` in the dict. Returns `{old_sha, new_sha}` on success. Also refreshes `source_hash` from disk. |
| `TestCheckStaleGitAware` | `check_stale(net, db_dir, git_aware=True)` | Returns `(results, upgraded, sha_bumped)` triple. With `git_aware`: fresh if no commits since `pinned_sha`; stale if content changed; **auto-bumps `pinned_sha`** to new HEAD when content hash is unchanged (commit churn but no content change). Falls back to hash-only staleness when `pinned_sha` is absent. |
| `TestParseDiffHunks` | `parse_diff_hunks(diff_text)` | Parses unified-diff hunk headers into `[(start, end), ...]` using the **old-file line numbers**. `@@ -89,5 +89,7 @@` → `(89, 93)`; `@@ -42 +42,2 @@` → `(42, 42)` (count defaults to 1); `@@ -89,0 @@` → excluded (zero-width hunks). Returns `[]` for empty/no-header input. |
| `TestFileLinesChangedSince` | `file_lines_changed_since(path, sha, start, end)` | True iff a hunk intersects `[start, end]`. Handles both committed changes (via `git diff sha..HEAD`) and **uncommitted working-tree** changes. **Fails open** — a bogus SHA returns `True` (assume changed). |
| `TestPinLines` | `pin_lines(net, node_id, start, end, db_dir)` | Sets `metadata["pinned_lines"] = "start-end"`. Auto-pins a SHA if one wasn't already set (`auto_pinned: True`). Preserves an existing SHA. Raises `ValueError` for `start < 1` or `start > end` and for a node with no `source`; raises `KeyError` for an unknown node. |
| `TestCheckStaleLineRange` | `check_stale` with `pinned_lines` set | When `pinned_lines` is present and git-aware, stale detection is **scoped**: a change outside the pinned band leaves the belief fresh; a change inside it marks it `content_changed`. When `pinned_lines` is absent, any file-level change is stale. Auto-bump preserves `pinned_lines` and refreshes `source_hash`. |

---

## Patterns

- **Class-per-function grouping** — pytest still collects them flatly, but the class boundary documents the SUT under test.
- **Fixture-injected real git** — `subprocess.run(["git", ...])` against `tmp_path` rather than mocking git. This is deliberate: pinning logic hinges on real `rev-parse`, `log`, and `diff` outputs, and the test suite exercises the exact shell semantics.
- **Assert both return value and durable side-effect** — every `pin_sources`/`pin_lines`/`pin_update` test verifies the returned dict *and* the mutated `node.metadata[...]` / `node.source_hash`. This is the signature of a pattern where the function is a "return a report, mutate the project" operation.
- **Error-as-dict in bulk path, exceptions in per-node path** — `pin_update` returns `{"error": ...}` (batch-friendly), while `pin_lines` raises `ValueError`/`KeyError` (single-call, precondition-checked). This split is intentional and the tests lock it down.
- **Fail-open on stale detection** — `file_lines_changed_since` with a bad SHA returns `True` (caller treats as "changed"), so a corrupt pin does not silently mask drift.
- **`_`-prefixed private helpers** outside the classes to avoid fixture/test-collection noise.

---

## Dependencies

**Imported by this file (SUT):**
- `reasons.network.Network` — the in-memory belief graph. Tests construct a `Network()`, call `net.add_node(id, text, source=..., source_hash=..., source_url=...)`, then mutate `net.nodes[id].metadata` directly.
- `reasons.check_stale.{check_stale, get_file_commit_sha, file_changed_since, file_lines_changed_since, hash_file, parse_diff_hunks, pin_lines, pin_source_url, pin_sources, pin_update}` — the ten functions under test.

**Standard library:** `hashlib`, `subprocess`, `pathlib.Path`, `pytest`.

**Imported-by:** nothing. This is a terminal test module; it is picked up by pytest discovery from the `tests/` directory. Changes here cannot break any other module.

**Implicit dependency on the `git` CLI** being on `PATH`; every test class except `TestParseDiffHunks` and `TestPinSourceUrl` requires it. A missing/old `git` will fail the `git_repo` fixture (`check=True`).

---

## Flow

Take the most representative path — `test_auto_bumps_sha_when_content_unchanged`:

1. `git_repo` fixture materializes a repo + `source.md` at HEAD.
2. Test captures `sha = _get_head_sha(git_repo)` and `content_hash = hash_file(git_repo / "source.md")`.
3. An in-memory `Network` is built with one node carrying `source="source.md"` and that hash; `pinned_sha` is set directly on metadata (bypassing `pin_sources`, which is fine because pinning is just metadata).
4. `_commit_change` twice: change → revert → so HEAD moves but the file bytes are byte-identical to the pin.
5. `check_stale(net, db_dir=git_repo, git_aware=True)` is called:
   - computes `new_head = _get_head_sha(...)` and compares to `pinned_sha` → different.
   - recomputes `hash_file(...)` → matches `node.source_hash`.
   - therefore: not stale, but the pin is out of date → **mutates** `metadata["pinned_sha"]` to `new_head`, increments `sha_bumped`, preserves `pinned_lines`.
6. Assertions check `results == []`, `upgraded == 0`, `sha_bumped == 1`, and `metadata["pinned_sha"] == new_head`.

Contrast with `test_falls_back_to_hash_without_pinned_sha`: no `pinned_sha`, so the git-aware branch short-circuits to plain hash comparison and a content change becomes `reason == "content_changed"`.

And `TestCheckStaleLineRange.test_fresh_when_change_outside_pinned_lines`: `pinned_lines = "1-2"`; line 5 changes; `file_lines_changed_since` returns `False` for `[1, 2]`; `check_stale` reports fresh — **even though the file hash changed**. This is the key semantic: line-scoped pinning deliberately ignores off-slice drift.

---

## Invariants

- **`pinned_sha` is a 40-char full SHA** — asserted as `len(...) == 40` in multiple places (`TestGetFileCommitSha`, `TestPinSources`, `TestPinUpdate`, `TestPinLines`).
- **Pin state lives in `node.metadata`**, not on the node itself, except `source_hash` which is a first-class attribute (`node.source_hash`).
- **`pin_sources` is idempotent by default** — running it twice without `force=True` produces zero results the second time (`test_skips_already_pinned`).
- **`pin_sources` skips `truth_value == "OUT"`** — pinned-but-out beliefs are not re-pinned (`test_skips_out_nodes`). Similarly they are skipped when `source` is `None` or the file is on disk but missing.
- **`pin_sources` backfills `source_hash` even when it's `""`** — the pin either pins SHA *or* ensures the content hash is current (`test_backfills_source_hash`).
- **`pin_lines` ordering:** `start >= 1` and `start <= end`; otherwise `ValueError`. Unknown node → `KeyError`. Both are hard preconditions, not soft errors.
- **`pin_lines` preserves an already-set `pinned_sha`** (`test_preserves_existing_sha`) — only auto-pins when the metadata key is absent.
- **`parse_diff_hunks` uses old-file line numbers** and the end is `start + count - 1` (inclusive). Zero-width hunks (`-89,0`) are excluded.
- **`pin_source_url` is a no-op outside `github.com`** — non-GitHub URLs must be left alone (`test_preserves_non_github_url`).
- **`file_lines_changed_since` fails open** — an invalid SHA returns truthy (reports stale rather than silently passing).
- **`check_stale` auto-bump preserves `pinned_lines`** (`test_auto_bumps_sha_preserves_pinned_lines`).
- **`check_stale` returns a 3-tuple** `(results, upgraded, sha_bumped)` — callers must unpack all three.

---

## Error Handling

Two distinct error philosophies are exercised:

1. **Batch/report style — `pin_update` and `pin_sources` never raise** for ordinary failure cases. They emit a payload dict whose `error` key encodes the reason:
   - `"not found"` — unknown node id.
   - `"no source"` — node has no `source` attribute.
   - `"source not found"` — source path is absent on disk.
   Tests assert these exact string literals (`test_error_on_missing_node`, `test_error_on_no_source`, `test_error_on_source_not_found`) — changing them is a contract break.

2. **Precondition style — `pin_lines` raises.** `ValueError` for invalid ranges (`start < 1`, `start > end`) and for nodes lacking a `source`; `KeyError` for unknown node ids. These are invoked with `pytest.raises(...)` in `TestPinLines`.

3. **Fail-open safe-default in `file_lines_changed_since`** — an unparseable SHA (e.g., all-zeros) causes the function to return `True`, i.e., "changed." This is the deliberate conservative choice to avoid missing drift when git history is unreadable.

4. **Subprocess failures bubble through the fixture**, not the SUT: `subprocess.run(..., check=True)` in `git_repo` and `_commit_change` raise `CalledProcessError` if git fails, which will surface as a test error (not a test failure), immediately pointing at environment problems.

---

## Topics to Explore

- [function] `reasons/check_stale.py:check_stale` — The orchestrator that returns `(results, upgraded, sha_bumped)`; the tests here only cover the git-aware path — read to see how non-git-aware mode, missing source files, and hash-only fallback are sequenced.
- [function] `reasons/check_stale.py:pin_source_url` — The regex/heuristic that turns `https://github.com/*/blob/<branch>/<path>` into a SHA permalink; the `source_path` disambiguation test is the key to understanding its branch-with-slash edge case.
- [file] `reasons/network.py` — `Network.add_node` and the `Node` shape; the tests mutate `node.metadata` and `node.source_hash` directly, so the serialization/persistence contract lives here.
- [file] `tests/conftest.py` — Any project-wide fixtures that could interact with the local `git_repo` fixture (e.g., a global `Network` or tmp-dir patching).
- [general] `metadata-persistence` — How `pinned_sha`, `pinned_lines`, and `verified_at` survive a save/reload cycle in storage/`pg.py`; the tests only check in-memory state.

---

## Beliefs

- `pin-sources-skips-pinned-unless-forced` — `pin_sources(net, ...)` returns zero results when every eligible node already has `metadata["pinned_sha"]`, unless `force=True` is passed.
- `pin-update-never-raises` — `pin_update` reports failures via the `"error"` key in each result dict (`"not found"`, `"no source"`, `"source not found"`) and does not raise for these cases.
- `pin-lines-raises-on-bad-range` — `pin_lines` raises `ValueError` when `start < 1` or `start > end`, and `KeyError` when the target node id is absent from the network.
- `check-stale-returns-3-tuple` — `check_stale` always returns `(results, upgraded, sha_bumped)`; the third element counts auto-bumped SHAs for pinned beliefs whose content is byte-identical.
- `line-scoped-pinning-ignores-off-slice-drift` — When `metadata["pinned_lines"]` is set, `check_stale(git_aware=True)` reports the belief fresh if the only commits touch lines outside the pinned inclusive range.
- `parse-diff-hunks-ends-inclusive` — `parse_diff_hunks("@@ -89,5 +89,7 @@")` returns `[(89, 93)]` (start + count − 1) and excludes zero-count hunks like `@@ -89,0 @@`.
- `pin-source-url-github-only` — `pin_source_url` rewrites only `github.com` URLs; non-GitHub URLs are returned unchanged regardless of the SHA argument.
