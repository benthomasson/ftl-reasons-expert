---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: get_file_commit_sha in reasons/check_stale.py

## `get_file_commit_sha`

### Purpose

`get_file_commit_sha` answers the question: “What is the most recent Git commit SHA that touched this file in the current branch history?”

It exists to support Git-aware source pinning and staleness checks. Callers use the returned SHA to store or update `node.metadata["pinned_sha"]`, so later checks can avoid/content-hash work when the source file has not changed since that commit.

It is a thin wrapper around:

```bash
git log -1 --format=%H -- <basename>
```

run from the file’s parent directory.

---

### Contract

**Preconditions**

- `filepath` is a `Path`.
- `filepath.parent` exists and is usable as a process working directory.
- Git is available in `PATH`.
- The parent directory is expected to be inside a Git work tree, though this is not enforced.
- The file is expected to be tracked by Git, though this is not enforced.

**Postconditions**

- On success, returns the stripped stdout of `git log -1 --format=%H -- <filepath.name>`.
- If Git exits non-zero or prints no stdout, returns `None`.
- Does not mutate the filesystem or the `Path` object.

**Invariants**

- It only passes the file’s basename to Git, not the full path.
- It runs Git with `cwd=filepath.parent`.
- It checks the current branch’s `HEAD` history by default; it does not use `--all`.
- It does not use `--follow`, so it does not follow renames across history.
- It does not inspect uncommitted changes.

### Assumptions not enforced by the type system

- `filepath.name` is the correct Git pathspec relative to `filepath.parent`.
- `filepath.parent` is inside a Git repository.
- The returned stdout is a valid commit SHA. The function does not validate length or hex format.
- The file has not been renamed in a way where the caller expects rename-following.
- The repository uses the expected Git history; uncommitted working-tree changes are invisible to this function.
- `subprocess.run` will not raise unexpected OS/decode errors.

---

### Parameters

- `filepath: Path`
  - The source file whose last touching commit should be found.
  - Only `filepath.name` and `filepath.parent` are used.
  - If `filepath` is relative, `cwd` is also relative to the current process working directory.
  - If `filepath.parent` does not exist, `subprocess.run` can raise `FileNotFoundError` / `NotADirectoryError`.
  - If `filepath` points to a directory, Git may treat the basename as a directory pathspec and return a commit touching files under it. This is not the intended usage.
  - If the file is untracked but exists, Git usually exits `0` with empty stdout, producing `None`.

---

### Return Value

Returns `str | None`.

- `str`: the stripped stdout from `git log -1 --format=%H -- <filepath.name>`, normally a full commit hash such as `a1b2c3...`.
- `None`:
  - Git exits with a non-zero return code, or
  - Git exits successfully but stdout is empty after stripping.

Callers must treat `None` as “SHA unavailable” rather than a single specific reason. It could mean:
- the directory is not a Git repository,
- the file has no commit history,
- the file is untracked,
- Git failed for another reason,
- the file has no reachable commits on the current branch.

The function does not validate that a returned string is actually 40 hex characters, so callers should assume it is whatever Git emitted.

---

### Algorithm

1. Run:

   ```python
   subprocess.run(
       ["git", "log", "-1", "--format=%H", "--", str(filepath.name)],
       capture_output=True,
       text=True,
       cwd=str(filepath.parent),
       timeout=30,
   )
   ```

2. `git log -1` asks for only the most recent commit.
3. `--format=%H` asks for the full commit hash.
4. `--` separates Git options from the pathspec, so filenames beginning with `-` are still treated as pathspecs.
5. Git is run from `filepath.parent`, and the pathspec is only `filepath.name`. This makes the pathspec relative to that parent directory.
6. If `returncode == 0` and `stdout.strip()` is non-empty, return the stripped stdout.
7. Otherwise return `None`.

---

### Side Effects

- Spawns a synchronous external `git` process.
- Reads Git repository metadata/history.
- Captures stdout and stderr.
- Does not modify files, Git state, or the filesystem.
- Can be relatively expensive because it invokes a new Git process per call.
- Has a 30-second timeout.

---

### Error Handling

- Non-zero Git exit code is swallowed and converted to `None`.
- Empty Git stdout is converted to `None`.
- `subprocess.TimeoutExpired` is **not** caught and will propagate.
- `FileNotFoundError` / `OSError` from missing `git`, invalid `cwd`, permission issues, etc. are **not** caught and will propagate.
- Decoding errors from `text=True` are not handled.
- No logging or diagnostics are emitted for Git failures.

This means callers get a mixed error model:

- Git command-level failures generally become `None`.
- Process-level failures generally become exceptions.

---

### Usage Patterns

This function is used by the Git-aware pinning and staleness code:

- `pin_sources(...)`: resolves each source path, calls `get_file_commit_sha(path)`, and skips the node if the result is `None`.
- `pin_update(...)`: calls it for explicitly selected nodes and reports `"not in git repo"` if `None` is returned.
- `pin_lines(...)`: calls it to auto-pin a SHA if one is not already present.
- `check_stale(...)`: when `git_aware=True`, calls it to bump `pinned_sha` when the file content hash is unchanged but the file’s last commit SHA differs from the stored pin.

Caller obligations:

- Resolve and validate the source path before calling.
- Handle `None`.
- Be prepared for `subprocess` exceptions to propagate.
- Avoid assuming that `None` means the file was deleted; deletion is handled elsewhere via path resolution.
- Consider batching/caching if checking many files in the same repository, because each call launches a separate Git process.

---

### Dependencies

- Python standard library `subprocess`.
- Python standard library `pathlib.Path`.
- External `git` executable from `PATH`.
- No internal module dependencies inside this function itself.

---

## Topics to Explore

- [file] `reasons/check_stale.py` — See how `get_file_commit_sha` participates in the full Git-aware staleness and pinning flow.
- [function] `reasons/check_stale.py:file_changed_since` — Companion check that detects committed and uncommitted changes since a pinned SHA.
- [function] `reasons/check_stale.py:file_lines_changed_since` — Line-level staleness logic that uses Git diffs instead of whole-file hashes.
- [function] `reasons/check_stale.py:pin_sources` — Shows how the returned SHA is persisted into node metadata.
- [general] `git-pathspec-relative-to-cwd` — Explains why running Git in `filepath.parent` while passing only `filepath.name` targets the intended file.

## Beliefs

- `get_file_commit_sha-requires-git-executable` — It shells out to `git`; if `git` is unavailable, `subprocess.run` can raise `FileNotFoundError` instead of returning `None`.
- `get_file_commit_sha-returns-none-on-nonzero-git-status` — A non-zero `git log` exit status is converted to `None`.
- `get_file_commit_sha-uses-basename-with-parent-as-cwd` — It runs Git in `filepath.parent` and passes only `filepath.name` after `--`.
- `get_file_commit_sha-does-not-catch-timeout-expired` — The 30-second `subprocess.run` timeout raises `subprocess.TimeoutExpired`; it is not converted to `None`.
- `get_file_commit_sha-enables-pinned-sha-updates` — Callers use its return value to set or bump `node.metadata["pinned_sha"]` during pinning and Git-aware staleness checks.
