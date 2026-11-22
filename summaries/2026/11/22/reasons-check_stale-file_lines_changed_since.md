---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: file_lines_changed_since in reasons/check_stale.py

## `file_lines_changed_since`

### Purpose
A "line-scoped" staleness check. Given a file, a commit SHA, and a 1-based line range, it asks git whether **any of those specific lines** were modified between that commit and the current working tree (committed **or** uncommitted). It exists to support a fast path in `check_stale` when a node has `pinned_sha` + `pinned_lines` metadata: if the referenced lines didn't change, the expensive full-file SHA-256 can be skipped entirely.

The general contract is *conservative*: `True` means "possibly changed, do the full content hash," `False` means "definitely unchanged within this range, safe to trust."

### Contract
- **Preconditions (not type-enforced):**
  - `filepath.parent` is inside a git working tree that contains `filepath`.
  - `since_sha` is a resolvable ref in that repo (full or short SHA); if not, `git diff` exits non-zero and the function returns `True`.
  - `1 <= line_start <= line_end`, and both refer to **1-based** line numbers on the *old* side of the diff (the file as it existed at `since_sha`).
- **Postconditions:**
  - Returns `True` iff any diff hunk overlaps `[line_start, line_end]` *on the old side*, or if git failed/timed out.
  - Returns `False` iff the working tree is identical to `since_sha` on those old lines.
- **Invariant:** The comparison is anchored to the *old* side of the diff so that line numbers match the stored `pinned_lines` (which were captured against `pinned_sha`).

### Parameters
| Name | Type | Meaning | Edge cases |
|---|---|---|---|
| `filepath` | `Path` | Target file. Only `.name` is passed to git; `.parent` is used as `cwd`. | Must be inside a git worktree; symlinks/odd basenames may break path matching. |
| `since_sha` | `str` | Ref to diff against (typically the node's `pinned_sha`). | Invalid SHA → non-zero exit → `True`. |
| `line_start` | `int` | First line of the range (inclusive, 1-based, old side). | No validation; if `> line_end`, overlap check is always `False`. |
| `line_end` | `int` | Last line of the range (inclusive, 1-based, old side). | Same. |

### Return Value
- `True` — content in `[line_start, line_end]` was touched (or the diff couldn't be trusted: non-zero exit, timeout).
- `False` — no hunk overlaps the range; caller may skip content hashing.

Callers **must** treat `True` as *conservative*, not authoritative — it forces a fallback to `hash_file`. Only `False` carries the "unchanged" guarantee.

### Algorithm
1. `subprocess.run(["git", "diff", "-U0", since_sha, "--", filepath.name], cwd=filepath.parent, timeout=30)`.
   - `-U0` emits zero context, so each hunk contains only the changed lines.
   - Diffing `since_sha` against the working tree captures both committed commits after `since_sha` **and** uncommitted (staged/unstaged) edits in a single invocation.
2. If `returncode != 0` → return `True` (defensive).
3. Parse `@@ -S[,C] +N[,K] @@` headers via `parse_diff_hunks(result.stdout)`, yielding **old-side** `(start, end)` ranges per hunk.
   - Hunks with old-side `count == 0` (pure insertions) are skipped, so inserting lines *above* the range doesn't falsely trigger a change.
4. For each `(hunk_start, hunk_end)`, check inclusive overlap: `hunk_end >= line_start and hunk_start <= line_end`. Any overlap → `True`.
5. No overlap → `False`.
6. `subprocess.TimeoutExpired` → `True`.

### Side Effects
- One `git` subprocess invocation (read-only). No filesystem writes, no state mutations.
- Relies on `cwd` being the file's parent — git resolves the worktree by walking upward.

### Error Handling
- **Handled → `True`:** non-zero exit codes (invalid ref, not a git repo, path issues surfaced by git) and `TimeoutExpired`.
- **Not handled → propagates:** `FileNotFoundError` if `git` is not installed, if `cwd` doesn't exist, or malformed `filepath`/`since_sha` that `subprocess` itself rejects. The docstring's "returns True on git errors" is only literally true for `TimeoutExpired` and non-zero exits.
- `parse_diff_hunks` uses regex; it won't raise on unexpected input, it just returns `[]`.
- Silent assumption: `line_start <= line_end`. If violated, the overlap predicate returns `False` (no exception), which could misclassify staleness.

### Usage Patterns
Called exactly once, inside `check_stale`:
```python
if not file_lines_changed_since(path, pinned_sha, ls, le):
    new_sha = get_file_commit_sha(path)
    if new_sha and new_sha != pinned_sha:
        # bump pinned_sha, verified_at, source_hash; sha_bumped += 1
    continue   # skip content hashing
```
Caller obligation: `pinned_lines` must have been validated when set (see `pin_lines`, which enforces `1 <= start <= end`). If `file_lines_changed_since` returns `True`, the code falls through to the ordinary `hash_file` comparison path.

### Dependencies
- **Internal:** `parse_diff_hunks` (sibling in this module).
- **Stdlib:** `subprocess`, `pathlib.Path`.

### Assumptions Not Enforced by Types
- `filepath.parent` is inside the git repo that owns `filepath`.
- `since_sha` is an ancestor of `HEAD` (or at least a valid ref); otherwise `git diff` fails and the function degrades to `True` — which is safe but means the "fast path" is effectively disabled.
- Only `filepath.name` is matched, so files with identical basenames from different directories cannot be disambiguated (fine here because `cwd = parent`).
- The caller already checked that `pinned_lines` and `pinned_sha` exist and parsed successfully (`check_stale` does this with a `try/except` around `split("-")`).

## Topics to Explore
- [function] `reasons/check_stale.py:parse_diff_hunks` — Converts `@@` headers into old-side `(start, end)` ranges and skips pure insertions; the correctness of the overlap test depends on it.
- [function] `reasons/check_stale.py:check_stale` — The only caller; shows how `False` result skips `hash_file` and potentially bumps `pinned_sha`.
- [function] `reasons/check_stale.py:pin_lines` — Establishes the `pinned_lines`/`pinned_sha` invariant (1 <= start <= end) that this function relies on.
- [function] `reasons/check_stale.py:file_changed_since` — The coarser whole-file variant; contrast how it handles uncommitted changes vs. this line-scoped check.
- [general] `git-diff-old-vs-new-side` — Understanding why old-side line numbers are the correct anchor for `pinned_lines`, and why `-U0` + count=0 hunk skipping gives insertion-insensitivity.

## Beliefs
- `file_lines_changed_since-is-conservative-on-error` — Returns `True` on non-zero git exit codes and `TimeoutExpired`, forcing content-hash fallback rather than risking a missed staleness.
- `file_lines_changed_since-uses-old-side-line-numbers` — Hunk ranges come from the old side of the diff, matching line numbers stored in `pinned_lines`.
- `file_lines_changed_since-skips-pure-insertions` — Because `parse_diff_hunks` discards hunks with old-side count=0, insertions above the range do not falsely invalidate pinned lines.
- `file_lines_changed_since-enables-pinned-lines-fast-path` — A `False` return lets `check_stale` skip `hash_file` and optionally bump `pinned_sha` without recomputing the source hash.
- `file_lines_changed_since-requires-git-worktree-parent` — It relies on `filepath.parent` being inside a git working tree; only `filepath.name` is passed to git, and `FileNotFoundError` from a missing git/cwd is not caught.
