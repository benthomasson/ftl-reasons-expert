---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Understanding why old-side line numbers are the correct anchor for `pinned_lines`, and why `-U0` + count=0 hunk skipping gives insertion-insensitivity.

# Why old-side line numbers anchor `pinned_lines`, and why `-U0` + count=0 skipping buys insertion-insensitivity

## 1. What `pinned_lines` actually stores

`pinned_lines` is written by `pin_lines` (`reasons/check_stale.py:436–470`) as a `"start-end"` string, e.g. `"89-93"` (see the assertion in `tests/test_pin_sources.py::TestPinLines.test_sets_pinned_lines`). At the moment of pinning, `pin_lines` also guarantees a `pinned_sha` — either one supplied by the caller or backfilled from `git log -1` (see `TestPinLines.test_auto_pins_sha_if_missing`).

So the pair **(`pinned_sha`, `pinned_lines`)** is the invariant: *"at commit `pinned_sha`, lines *start…end* of the source file are the material this belief rests on."* The line numbers are not positions in the *current* file — they are positions in the file **as it existed at `pinned_sha`**.

## 2. Why the OLD side of the diff is the correct side to test against

When `check_stale` decides whether to short-circuit, it dispatches into `file_lines_changed_since` (`reasons/check_stale.py:273–292`), which runs:

```
git diff -U0 since_sha -- <file>
```

The header of every hunk in that diff carries **two** coordinate systems:

```
@@ -<old_start>,<old_count> +<new_start>,<new_count> @@
```

- **OLD side** = the file at `since_sha` (i.e. at `pinned_sha`).
- **NEW side** = the working-tree file after any commits and uncommitted edits.

Because `pinned_lines` was recorded against the file *at* `pinned_sha`, the only side whose numbering matches is the OLD side. The docstring on `file_lines_changed_since` states this explicitly:

> "Diffs since_sha against the working tree (committed + uncommitted) in a single call. Uses the OLD side of the diff (stored lines reference the file at pinned_sha)."

If we used the NEW side, we would be asking "do lines *start…end* of the current file contain changes?" That is a positional question, not an identity question: new-side numbering has been shifted by every insertion/deletion that happened since `pinned_sha`, so the numbers no longer point at the pinned material.

`parse_diff_hunks` (`reasons/check_stale.py:251–263`) enforces this by construction — the regex captures only `-(\d+)(?:,(\d+))?` and ignores the `+…` side entirely:

```python
re.finditer(r'^@@ -(\d+)(?:,(\d+))? \+\d+(?:,\d+)? @@', ...)
```

The overlap test in `file_lines_changed_since` then compares that OLD-side `(hunk_start, hunk_end)` against the stored `(line_start, line_end)`:

```python
if hunk_end >= line_start and hunk_start <= line_end:
    return True
```

## 3. What `-U0` buys you

The default `git diff` emits 3 lines of context on each side. That widens every hunk header to include unchanged neighbors. For example, editing one line of a 5-line change would produce `@@ -87,7 +87,7 @@` — the reported range contains lines that never changed. Because `file_lines_changed_since` uses that range as the "changed interval," any pinned range touching those context lines would be falsely flagged stale.

`-U0` collapses context to zero. Every hunk header now reports **only the lines actually modified/deleted/inserted**, e.g. `@@ -42,1 +42,1 @@`. The overlap test then only triggers on genuinely changed old-side content. `TestParseDiffHunks.test_parses_single_hunk` (`tests/test_pin_sources.py`) pins this contract: `@@ -89,5 +89,7 @@` yields `(89, 93)`.

## 4. Why "skip count == 0" gives insertion-insensitivity

A **pure insertion** under `-U0` produces a hunk whose OLD count is 0:

```
@@ -89,0 +90,3 @@     ← 3 new lines inserted between old lines 88 and 89
```

Zero old lines are affected — nothing in the anchored file was modified or deleted; only new material was added. The old-side "range" would be the degenerate point `(89, 88)`.

If `parse_diff_hunks` kept it (start = 89, count = 0, end = 88), the overlap test would be evaluated against a *point* rather than a range, and it can and does produce false positives. For example, with `pinned_lines = "89-93"`:

```
hunk_end   = 88  → 88 >= 89 ?  false  →  no match   (OK)
```

but for an insertion just inside the pinned range:

```
@@ -90,0 +90,3 @@     ← insert before old line 90
hunk_end   = 89  → 89 >= 89 ? true
hunk_start = 90  → 90 <= 93 ? true
→ FALSE POSITIVE
```

The pinned lines 89–93 are physically unchanged; they just shifted to new positions. The correct call is "fresh." Skipping `count == 0` avoids this:

```python
if count == 0:
    continue
hunks.append((start, start + count - 1))
```

Skipping also handles the extreme case of `@@ -0,0 +1,N @@` (file created from nothing) — no old lines exist to be stale.

**Deletions are the mirror case**: a pure deletion looks like `@@ -50,3 +49,0 @@` — old count is 3, so `parse_diff_hunks` keeps it and returns `(50, 52)`. That correctly flags `pinned_lines` that overlapped the deleted region. `TestFileLinesChangedSince.test_detects_change_in_pinned_range` covers modification; `TestCheckStaleLineRange.test_stale_when_change_inside_pinned_lines` covers the end-to-end stale path.

## 5. The full flow

1. `check_stale` (`reasons/check_stale.py:107–120`) reads `pinned_lines`, splits on `-`, and calls `file_lines_changed_since`.
2. If `pinned_lines` is malformed, it `pass`es through to whole-file content hashing (`TestCheckStaleLineRange.test_falls_back_to_whole_file_without_pinned_lines`).
3. `file_lines_changed_since` runs `git diff -U0 since_sha -- file`, parses OLD-side hunk ranges, and returns True iff any hunk overlaps the anchored range.
4. On "no overlap", `check_stale` bumps `pinned_sha` to the current HEAD and refreshes `source_hash` (auto-bump path — see `TestCheckStaleLineRange.test_auto_bumps_sha_preserves_pinned_lines`). The `pinned_lines` value itself is deliberately preserved, which is coherent precisely because it stays anchored to whatever is now the pinned file content.
5. On "overlap" or "malformed", control falls through to `hash_file(.md)` and reports `reason: "content_changed"` if the SHA-256 differs.

Git failures (bad SHA, timeout) return `True` from `file_lines_changed_since`, which forces the conservative whole-file hash fallback (`TestFileLinesChangedSince.test_fails_open_on_bad_sha`).

## 6. Net invariant

- **`pinned_lines` ↔ OLD side of `git diff since_sha`** — same coordinate system, because both describe the file at pin time.
- **`-U0`** — hunk ranges reflect only modified/deleted lines, not neighbors.
- **`count == 0` skip** — pure insertions are invisible to the overlap test, so shifting the file around never falsely invalidates a pinned range.
- **Deletions and modifications (old count > 0)** — remain detectable, which is exactly the class of changes a pinned claim cares about.

## Topics to Explore

- [function] `reasons/check_stale.py:file_lines_changed_since` — Verify the overlap predicate for boundary cases (empty files, hunks starting at line 1, insertions immediately adjacent to the pinned range).
- [function] `reasons/check_stale.py:check_stale` — Trace the auto-bump path (lines 113–120) and what state is preserved vs. refreshed when lines are unchanged but HEAD moved.
- [file] `tests/test_pin_sources.py` — `TestParseDiffHunks` and `TestFileLinesChangedSince` act as the executable spec for the OLD-side and count=0 contracts.
- [function] `reasons/check_stale.py:pin_lines` — How `pinned_lines` validation and auto-pin of `pinned_sha` interact; confirms the (sha, range) invariant at write time.
- [general] `git-diff-hunk-header-semantics` — The unified diff spec for `-A,B +C,D` counts, especially what a zero count means for the anchor side.

## Beliefs

- `pinned-lines-are-anchored-to-pinned-sha` — `pinned_lines` values reference line numbers in the file at `pinned_sha`, which is why `file_lines_changed_since` must test the OLD side of `git diff since_sha`.
- `parse-diff-hunks-uses-only-old-side` — `parse_diff_hunks` captures `-start,count` and ignores the `+start,count` side of each hunk header (regex at `reasons/check_stale.py:259`).
- `u0-eliminates-context-false-positives` — Running `git diff -U0` narrows hunk headers to changed lines only, so the overlap test is not polluted by surrounding context lines.
- `count-zero-hunk-skip-prevents-insertion-false-positives` — Skipping hunks with old count == 0 in `parse_diff_hunks` means pure insertions never overlap the pinned range, giving insertion-insensitivity while still detecting modifications and deletions.
- `malformed-pinned-lines-fall-through-to-content-hash` — When `pinned_lines` cannot be parsed, `check_stale` skips the range test and falls back to whole-file SHA-256 comparison.
