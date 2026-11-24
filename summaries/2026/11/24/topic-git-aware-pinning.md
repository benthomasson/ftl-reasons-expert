---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: The interaction of `pinned_sha` and `pinned_lines` and why the pinned-lines path bumps `source_hash` with a full-file hash even when only a range is relevant.

# `pinned_sha` × `pinned_lines`: how they cooperate (and why `source_hash` gets a full-file bump)

## Data model

- `source_hash` — a **full-file SHA-256** (`reasons/check_stale.py:15-17`, `hash_file`). It is the canonical record for content-hash mode: `check_stale` compares `hash_file(path)` against this value to detect `content_changed` (`check_stale.py:~135-145`).
- `pinned_sha` — the git commit that last touched the source file, used as an anchor when `git_aware=True` (`check_stale.py:105-106`).
- `pinned_lines` — a range like `"89-93"`, stored as text (`reasons/storage.py:88`, `check_stale.py:453`).
- `pinned_sha` and `pinned_lines` are also denormalized into the `node_sources` table (`reasons/storage.py:177-186`), but `check_stale` reads them from `node.metadata`.

The pairing is set together by `pin_lines` (`check_stale.py:436-469`), which auto-pins a SHA if none exists yet — so `pinned_lines` *implies* `pinned_sha` after that call.

## The check_stale flow

In `reasons/check_stale.py:105-123`:

```python
pinned_sha = node.metadata.get("pinned_sha") if node.metadata else None
if git_aware and pinned_sha:
    pinned_lines = node.metadata.get("pinned_lines") if node.metadata else None
    if pinned_lines:
        try:
            parts = pinned_lines.split("-")
            ls, le = int(parts[0]), int(parts[1])
        except (ValueError, IndexError):
            pass                       # line 113 — malformed → fall through
        else:
            if not file_lines_changed_since(path, pinned_sha, ls, le):
                new_sha = get_file_commit_sha(path)           # 116
                if new_sha and new_sha != pinned_sha:
                    node.metadata["pinned_sha"] = new_sha     # 118
                    node.metadata["verified_at"] = date.today().isoformat()
                    node.source_hash = hash_file(path)        # 120 (full-file SHA-256)
                    sha_bumped += 1
                continue
    elif not file_changed_since(path, pinned_sha):
        continue
```

Semantics of the pinned-lines branch:

1. **Fast path.** If `file_lines_changed_since(path, pinned_sha, ls, le)` returns `False`, no hunk between `pinned_sha..HEAD` overlaps `[ls, le]`. The node is *not stale* for its pinned range, even if the file has other changes. `continue` skips the whole-file hash comparison.
2. **Advance the anchor.** `get_file_commit_sha(path)` returns the new latest commit for that file. If it differs from `pinned_sha`, we advance `pinned_sha` so future diffs only need to inspect `new_sha..HEAD`.
3. **Refresh the full-file hash.** The same block re-hashes the whole file and stores it in `node.source_hash`.
4. `sha_bumped` is returned from `check_stale` as part of its `(results, upgraded, sha_bumped)` tuple so the caller persists all of this.

Contrast with the *whole-file* git-aware branch (`elif not file_changed_since(...)` at line 123) — no bump, because if no commit touched the file at all, `pinned_sha` is still the file's current commit.

Contrast with the **bottom-of-loop** bump at `check_stale.py:150-153`:

```python
elif git_aware and pinned_sha:
    new_sha = get_file_commit_sha(path)
    if new_sha and new_sha != pinned_sha:
        node.metadata["pinned_sha"] = new_sha
        node.metadata["verified_at"] = date.today().isoformat()
        sha_bumped += 1
```

That `elif` runs after `current_hash == node.source_hash`, i.e. **byte-identical content**. Here only `pinned_sha` is updated, never `source_hash` — because `source_hash` is already correct.

## Why the pinned-lines path *must* bump `source_hash`

There are two mutually-consistent invariants the whole-file hash is meant to uphold:

- **`source_hash` must equal `hash_file(path)` for any currently-fresh node** in content-hash mode.
- **`pinned_sha` must equal "the commit the box last verified against"** when the node is fresh in git-aware mode.

When `file_lines_changed_since(...)` returns `False` but `new_sha != pinned_sha`, the situation is:

- Commits *did* touch the file since `pinned_sha` — bytes outside `[ls, le]` may have changed.
- We still consider the node **fresh** because the pinned lines were untouched.
- Therefore `hash_file(path)` **may no longer equal** the old `node.source_hash`.

If we bumped only `pinned_sha` and left the old `source_hash` in place, `source_hash` would silently drift from on-disk reality. That drift is harmful because `source_hash` is the fallback:

- `file_lines_changed_since` **returns True on git errors** (docstring `check_stale.py:273`) to force content-hash fallback.
- Malformed `pinned_lines` (`check_stale.py:110-113`) also falls through to content hash.
- Any subsequent run with `git_aware=False` (`check_stale.py:~130-145`) hits `current_hash = hash_file(path)`, compares to `node.source_hash`, and would report a spurious `content_changed`.

By re-hashing the full file at `check_stale.py:120`, both pointers (`pinned_sha`, `source_hash`) stay synchronised with the current on-disk bytes. The hidden range-hash idea is deliberately *not* used — `pin_lines` never stores a hash of `[ls, le]`; it stores the *range* and lets `file_lines_changed_since` diff against the anchor going forward. So the only hash the system knows how to store is the full-file SHA-256, and it must be refreshed whenever the anchor is advanced across changed bytes.

The tests at `tests/test_pin_sources.py:427-460` (`test_fresh_when_change_outside_pinned_lines` / `test_stale_when_change_inside_pinned_lines`) codify this contract: outside-range edits stay fresh; inside-range edits go stale. The `source_hash` bump is exactly what makes that "fresh" outcome safe under the non-git-aware fallback too.

## Topics to Explore

- [function] `reasons/check_stale.py:file_lines_changed_since` — Core predicate that parses diff hunks against `[ls, le]`; also the git-error path (~line 273) that forces the content-hash fallback, which is exactly why `source_hash` must stay accurate.
- [function] `reasons/check_stale.py:pin_lines` — Establishes the `pinned_sha` ↔ `pinned_lines` pairing and the `auto_pinned` return value; the starting point of every invariant above.
- [file] `reasons/check_stale.py` — The whole `check_stale` function: fast paths, `sha_bumped` accounting, `source_deleted` handling, truncated-hash upgrade path (`test_pin_sources.py:1` returns the tuple shape).
- [file] `reasons/storage.py` — `node_sources` schema and the mirror-from-metadata insert (`storage.py:177-186`); useful for understanding which fields are source of truth vs. denormalized cache.
- [diff] `tests/test_pin_sources.py:427-460` — The `test_fresh_when_change_outside_pinned_lines` and `test_stale_when_change_inside_pinned_lines` cases pin the intended boundary semantics.

## Beliefs

- `pinned-lines-requires-git-aware-and-pinned-sha` — The range-check fast path only runs inside `if git_aware and pinned_sha:` (`reasons/check_stale.py:106`); without a `pinned_sha`, `pinned_lines` is inert.
- `pinned-lines-bump-both-pinned-sha-and-source-hash` — When `file_lines_changed_since` returns `False` and the file's HEAD commit advanced, `check_stale` writes both `node.metadata["pinned_sha"]` and `node.source_hash = hash_file(path)` (`check_stale.py:118-120`).
- `source-hash-is-full-file-sha256` — `hash_file` (`check_stale.py:15-17`) hashes the entire file's bytes with SHA-256; there is no range-hash primitive and `pinned_lines` stores only the "start-end" string.
- `sha-bump-without-source-hash-update-is-safe` — The trailing `elif git_aware and pinned_sha` (`check_stale.py:150-153`) updates only `pinned_sha` because it runs after `current_hash == node.source_hash`, i.e. bytes are unchanged.
- `pinned-lines-fall-back-to-source-hash` — Malformed `pinned_lines` (`check_stale.py:110-113`) and git errors inside `file_lines_changed_since` (docstring `check_stale.py:273`) both force content-hash comparison, which is only correct because the pinned-lines path keeps `source_hash` fresh.
