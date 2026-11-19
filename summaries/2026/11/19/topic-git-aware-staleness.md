---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: How `pinned_sha` + `pinned_lines` combine to avoid full-file hashing on every run

# How `pinned_sha` + `pinned_lines` avoid full-file hashing

## The expensive thing being avoided

Full-file hashing happens in `reasons/check_stale.py:5-7`:

```python
def hash_file(path: Path) -> str:
    return hashlib.sha256(path.read_bytes()).hexdigest()
```

It reads the entire file from disk and SHA-256s it. The goal of the pinning metadata is to prove, via cheap `git` subprocess calls, that this call is unnecessary on most runs.

## The main control flow

The relevant block is in `check_stale()` (`reasons/check_stale.py:105-155`):

```python
pinned_sha = node.metadata.get("pinned_sha") if node.metadata else None
if git_aware and pinned_sha:
    pinned_lines = node.metadata.get("pinned_lines") if node.metadata else None
    if pinned_lines:
        try:
            parts = pinned_lines.split("-")
            ls, le = int(parts[0]), int(parts[1])
        except (ValueError, IndexError):
            pass                     # malformed → fall through to hash
        else:
            if not file_lines_changed_since(path, pinned_sha, ls, le):
                new_sha = get_file_commit_sha(path)
                if new_sha and new_sha != pinned_sha:
                    node.metadata["pinned_sha"] = new_sha
                    node.metadata["verified_at"] = date.today().isoformat()
                    node.source_hash = hash_file(path)
                    sha_bumped += 1
                continue             # ← NO hash_file() on happy path
    elif not file_changed_since(path, pinned_sha):
        continue                     # ← NO hash_file() when only pinned_sha is set
```

The docstring at `reasons/api.py:2692` and `reasons/check_stale.py:74` states the invariant: *"nodes with pinned_sha skip content hashing when no commits touched the file since the pinned SHA."*

Two layers do the skipping.

## Layer 1 — `pinned_sha` alone (coarse)

If a node has only a `pinned_sha`, the code calls `file_changed_since(path, pinned_sha)` (`reasons/check_stale.py:234-260`). That function performs two `git` calls:

1. `git log --format=%H <since_sha>..HEAD -- <file>` — did *any* commit touch this file after the pin?
2. `git diff --quiet HEAD -- <file>` — are there *uncommitted* changes to it right now?

If both are clean, `check_stale` `continue`s without calling `hash_file`. If the file *was* touched, or if `git` returns an error (`return True` on failure per the docstring at line 272-273), the function falls through to the full-hash path.

## Layer 2 — `pinned_sha` + `pinned_lines` (fine-grained)

`pinned_lines` is a string like `"89-93"` describing the OLD-side line range of the file *as it existed at `pinned_sha`*. When present, it overrides Layer 1 via the `if pinned_lines:` branch (line 108).

`file_lines_changed_since(path, pinned_sha, ls, le)` (`reasons/check_stale.py:267-297`) runs:

```python
git diff -U0 <since_sha> -- <file>
```

A single subprocess call that diffs the *working tree* (committed + uncommitted) against the pinned commit. It then parses the hunk headers with `parse_diff_hunks` (line 262-277):

- The regex matches `@@ -<start>,<count> +... @@`, extracting **OLD-side** ranges — this is deliberate, because `pinned_lines` refers to the file at `pinned_sha`, not the current file.
- Hunks with `count == 0` (pure insertions that don't touch existing old lines) are skipped.

Then the intersection test (line 292-295):

```python
for hunk_start, hunk_end in parse_diff_hunks(result.stdout):
    if hunk_end >= line_start and hunk_start <= line_end:
        return True
```

The function returns `True` only when a diff hunk actually overlaps the pinned range. If the file *changed* but the edits land outside `ls..le`, `check_stale` still `continue`s and skips `hash_file`. That behaviour is pinned down by the tests:

- `tests/test_pin_sources.py:427` `test_fresh_when_change_outside_pinned_lines`
- `tests/test_pin_sources.py:442` `test_stale_when_change_inside_pinned_lines`

So the combine: **`pinned_sha` bounds *when* the check applies; `pinned_lines` narrows *where* within the file the change must have landed to invalidate the node.** Both must be negative (no change in scope) to short-circuit.

## Why the fast path is *sustainable* — the SHA bump

If the fast path just returned, `pinned_sha` would silently become stale; on the next commit to an unrelated part of the file, every future run would fall into "file touched since pin" and rehash.

That's why lines 116-122 exist:

```python
new_sha = get_file_commit_sha(path)
if new_sha and new_sha != pinned_sha:
    node.metadata["pinned_sha"] = new_sha
    node.metadata["verified_at"] = date.today().isoformat()
    node.source_hash = hash_file(path)   # ONE-TIME rehash to stay consistent
    sha_bumped += 1
continue
```

When the pinned lines are confirmed untouched but HEAD has advanced, the code advances `pinned_sha` to the current file-commit SHA and refreshes `source_hash` once. After that, the node goes back to the fast path. The `sha_bumped` counter is returned so the caller knows storage needs to be rewritten (`reasons/api.py:2707` `needs_write = upgrade_hashes or git_aware`). The test `test_auto_bumps_sha_when_content_unchanged` (`tests/test_pin_sources.py`) exercises exactly this.

Symmetrically, lines 148-155 handle the case where `hash_file()` *was* computed and matched `source_hash` but HEAD moved — the SHA is bumped without an extra rehash, since we just proved the content is identical.

## Fallback semantics (deliberate fail-safe)

Every git-driven short-circuit is designed to **fall back to full hashing when in doubt**:

- `file_lines_changed_since` returns `True` on non-zero exit **and** on `TimeoutExpired` (line 296-297) → forces `hash_file`.
- `file_changed_since` returns `True` on any git error.
- Malformed `pinned_lines` (`ValueError`/`IndexError` → `pass` at line 113) → falls through to `hash_file`.
- Missing `pinned_sha` (or `git_aware=False`) → skips the whole block entirely and hashes.

The only way a node avoids `hash_file` on a run is an *affirmative* signal from git that nothing relevant changed. Anything ambiguous is expensive-but-correct.

## Where the metadata is populated

You can see the storage layer is wired for both fields via `reasons/storage.py:88` (`pinned_lines TEXT DEFAULT '',`), `reasons/storage.py:177`, `reasons/storage.py:284`, `reasons/storage.py:312`, and the round-trip in `reasons/api.py:2274-2284` / `1833-1838`. Populating `pinned_sha` is `pin_sources` (referenced at `reasons/check_stale.py:320-361`), and `pinned_lines` is set by `pin_lines` (`reasons/check_stale.py:436-469`), which also auto-pins the SHA if not already set. The CLI exposes the result (`reasons/cli.py:540`, `reasons/cli.py:1248`).

## What the observations don't show

- The full bodies of `pin_sources`, `pin_lines`, and `pin_update` (lines 301-473 of `check_stale.py` are outside the excerpt — only their call sites and docstrings are visible).
- Whether `git_aware=True` is the default for any CLI entry point (only the API signature is visible).
- How `pinned_lines` is initially *computed* (e.g., is it user-supplied or derived from a `line_start`/`line_end` on the node?), beyond the fact that it lands as `"{line_start}-{line_end}"` (line 453).

Those would be worth confirming with the omitted code before relying on the feature.

## Topics to Explore

- [file] `reasons/check_stale.py` — Lines 301-473 contain `pin_sources`, `pin_update`, `pin_lines`, and the rest of the git-aware pin lifecycle that this explanation depends on
- [function] `reasons/check_stale.py:file_lines_changed_since` — The exact git-diff subprocess and hunk-intersection test that makes the fine-grained skip safe
- [function] `reasons/check_stale.py:parse_diff_hunks` — How OLD-side line ranges are extracted, and why `count == 0` hunks are skipped
- [file] `tests/test_pin_sources.py` — Behavioral contracts for every branch of the fast/slow path (`TestCheckStaleGitAware`, `TestPinSources`, `TestPinUpdate`)
- [function] `reasons/api.py:check_stale` — The public wrapper that turns on `git_aware`, computes `needs_write`, and persists bumped `pinned_sha` / `source_hash`

## Beliefs

- `pinned-lines-shortcircuit-hashing` — When `file_lines_changed_since` finds no overlapping hunk, `check_stale` `continue`s without calling `hash_file` (check_stale.py:115-122).
- `pinned-sha-has-coarse-file-fallback` — If `pinned_lines` is absent, `file_changed_since` is used instead and still avoids `hash_file` when no commit or working-tree change touches the file (check_stale.py:123).
- `malformed-pinned-lines-falls-back-to-hash` — A `pinned_lines` value that fails `int-int` parsing triggers the full-file hash path rather than silently succeeding (check_stale.py:110-113).
- `sha-bump-refreshes-source-hash` — In the pinned-lines fast path, when `pinned_sha` advances, `source_hash` is recomputed via `hash_file` so the stored pair stays consistent (check_stale.py:120).
- `git-errors-force-content-hash-fallback` — `file_changed_since` and `file_lines_changed_since` return `True` on git errors or timeouts, forcing the expensive hash path (check_stale.py:272-273, 279-297).
