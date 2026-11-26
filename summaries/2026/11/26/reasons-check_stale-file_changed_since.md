---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Function: file_changed_since in reasons/check_stale.py

# `file_changed_since`

## Purpose

Given a file path and a reference commit SHA, determine whether that file has any modification **since** that commit — either a *committed* change (a commit on `HEAD` after `since_sha` that touched the file) or an *uncommitted* change (staged or unstaged working-tree edits).

It exists as the foundation of the **git-aware staleness shortcut** in `check_stale`. When a node carries a `pinned_sha`, the checker wants to avoid rereading and re-hashing the source file if git can prove the file hasn't moved. This function is that proof. It is deliberately designed to **fail closed**: whenever git cannot answer the question confidently, it returns `True`, forcing the caller back to the slower but always-correct content-hash path.

## Contract

- **Precondition (assumed, not enforced):** `filepath` points at a file inside a git working tree; `filepath.parent` is a directory that git can resolve a repo from (git walks up from `cwd` to find `.git`).
- **Precondition (assumed):** `since_sha` names a commit reachable in the repo. If not, git fails and the function returns `True`.
- **Postcondition:**
  - Returns `False` **only** when both git commands succeeded *and* reported no changes. That is the sole "definitely unchanged" case.
  - Returns `True` in every other case: committed changes found, uncommitted changes found, nonzero exit codes, or timeouts.
- **Invariant:** the function never raises `subprocess.TimeoutExpired`. It *may* raise other exceptions (notably `FileNotFoundError` if the `git` binary is absent).
- **Invariant:** the function is pure with respect to program state — it does not mutate anything the caller holds.

## Parameters

- **`filepath: Path`** — Path to the source file. The function uses `filepath.parent` as the subprocess `cwd` and `filepath.name` (the basename) as the pathspec. Because `cwd` is the file's own parent directory, the basename is the correct relative path — but note this means **any subdirectory context above `filepath.parent` is discarded**. Git still resolves the repo by walking up from `cwd`, and the pathspec is relative to `cwd`, so this works. It breaks only if `filepath.parent` is not the direct directory containing the file (e.g., a caller passing a symlink-resolved path with a differing logical parent) — not enforced by the type system.
- **`since_sha: str`** — A commit SHA (could technically be any git rev). Assumed to be `pinned_sha`, i.e., the commit the node was last verified against. If it doesn't exist, the first git command fails → `True`.

## Return Value

A `bool`:

- `True` — "assume changed / can't tell" → caller must fall back to content hashing.
- `False` — "git affirmatively confirms no committed or uncommitted change to this file since `since_sha`" → caller may skip content hashing.

Callers must treat `True` as ambiguous (change *or* error) and MUST NOT interpret it as a confirmed mutation.

## Algorithm

1. **Committed-change probe.** Run:
   ```
   git log --format=%H {since_sha}..HEAD -- {filepath.name}
   ```
   with `cwd=filepath.parent`, `capture_output=True`, `text=True`, `timeout=30`.
   - Nonzero exit → `return True`.
   - Non-empty stdout → there is at least one commit reachable from `HEAD` that is not in `since_sha` and that touches the file → `return True`.
2. **Uncommitted-change probe.** Run:
   ```
   git diff --quiet HEAD -- {filepath.name}
   ```
   - `--quiet` exits `0` when the working tree matches `HEAD` for that path, `1` when there are staged/unstaged differences, `2+` on error.
   - Any nonzero exit → `return True`. This intentionally collapses "there are local edits" and "git errored" into the same `True`, which is safe for this use.
3. **Confirmed clean.** `return False`.

The whole body is wrapped in `try`/`except subprocess.TimeoutExpired: return True`.

## Side Effects

- Spawns **two** `git` subprocesses (sequentially, up to 60s worst-case) with `cwd` set to `filepath.parent`.
- Reads repository state (commit graph, index, working tree) — no writes.
- Captures stdout/stderr fully into memory.
- No mutation of `filepath`, `since_sha`, or any global/module state.

## Error Handling

- **`subprocess.TimeoutExpired`** — caught explicitly; returns `True`.
- **Git nonzero exit** (bad SHA, not a repo, path ambiguity, merge conflicts in the index) — not an exception; converted to `True` via the returncode checks.
- **`FileNotFoundError`** — raised if the `git` executable isn't on `PATH`. **Not caught.** The docstring's "Returns True on git errors" is narrower than it sounds: it covers git *command* failures and timeouts, not process-launch failures. In an environment without git installed, this function will raise, violating the fail-closed promise at that boundary.
- **`PermissionError` / other OSError** — likewise uncaught.

## Usage Patterns

Primary caller is `check_stale` when `git_aware=True`, a `pinned_sha` exists, and **no `pinned_lines`** is set:

```python
elif not file_changed_since(path, pinned_sha):
    continue
```

If it returns `True`, execution falls through to `hash_file(path)` and the full content comparison; if it returns `False`, the node is skipped entirely (treated as fresh). Callers are obligated to handle the `True`-means-"maybe" semantics by doing the content hash — the function is a *fast path only*.

## Dependencies

- Stdlib: `subprocess`, `pathlib.Path` (via the signature).
- External process: `git` CLI.
- No imports from other project modules; no coupling to `Network`, `hash_file`, etc.

## Assumptions Not Enforced by Types

1. `filepath.parent` is inside a git work tree (or a subdirectory whose ancestors include one).
2. `filepath.name` uniquely identifies the file relative to `filepath.parent` — true because `cwd` is that parent, but silently wrong if a caller passes a path whose `.parent` differs from the physical directory containing the file.
3. `since_sha` is a valid commit.
4. File is **tracked**. `git diff HEAD` does not report untracked files, so a newly-created-but-untracked file could be reported as "unchanged" (`False`) even though its content differs from `HEAD` — because there's nothing in `HEAD` to diff against. In practice nodes point at files that existed at some commit, so this is usually safe, but it isn't asserted anywhere.
5. Git is installed and executable.

## Topics to Explore

- [function] `reasons/check_stale.py:file_lines_changed_since` — The line-scoped variant; contrasts with this file-scoped check and shares the fail-closed pattern via `parse_diff_hunks`.
- [function] `reasons/check_stale.py:check_stale` — The orchestrator that decides when `file_changed_since` runs vs. full content hashing.
- [function] `reasons/check_stale.py:get_file_commit_sha` — Produces the `since_sha`/`pinned_sha` values this function consumes, using the same cwd/basename idiom.
- [file] `tests/test_check_stale.py` — Covers the git-aware branches, likely including the "returns True on git error" fallback path.
- [general] `git-fail-closed-pattern` — How this module consistently collapses ambiguous git failures into "assume stale" to preserve correctness.

## Beliefs

- `file-changed-since-is-a-fast-path` — The function is only a shortcut; callers must fall back to content hashing whenever it returns `True`.
- `file-changed-since-uses-basename-relative-to-parent` — Both git commands pass `filepath.name` and set `cwd=filepath.parent`, so the pathspec is the basename relative to its own directory.
- `file-changed-since-fails-closed-on-nonzero-exit` — Any nonzero return code from either git command is converted to `True`, not to an exception.
- `file-changed-since-does-not-catch-process-launch-errors` — Only `subprocess.TimeoutExpired` is caught, so `FileNotFoundError` from a missing `git` binary propagates.
- `file-changed-since-ignores-untracked-files` — `git diff --quiet HEAD` reports only tracked-file differences, so untracked files can be misclassified as unchanged.
