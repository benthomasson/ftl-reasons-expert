---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: parse_diff_hunks in reasons/check_stale.py

# `parse_diff_hunks` — Explanation

## Purpose

Parses the output of a unified diff (specifically the hunk header lines like `@@ -10,5 +10,7 @@`) and extracts the **OLD-side line ranges** that the diff touches.

It exists so that `file_lines_changed_since` can answer the question *"did any of the lines I pinned (e.g. `10-14`) get modified since commit X?"* by checking whether any old-side hunk range overlaps the pinned range. The OLD side is what matters here because the caller diffs `git diff -U0 <pinned_sha> -- file`, so "old" is the file as it existed when the lines were pinned.

## Contract

- **Precondition**: `diff_output` is (typically) the stdout of a git unified diff (`git diff -U0 ...`). It's tolerant of any string; it just extracts whatever matches.
- **Postcondition**: Returns a list of inclusive `(start, end)` integer pairs, one per *non-empty* old-side hunk, in the order the hunks appear in the diff.
- **Invariant**: For every returned pair `(s, e)`, `s >= 1` (for real diffs; smaller values are possible only with malformed input) and `e >= s` because `count == 0` hunks are filtered out. `count == 0` would otherwise produce `(s, s-1)` — an inverted/empty range — which is precisely why the skip exists.

## Parameters

| Name | Type | Meaning |
|---|---|---|
| `diff_output` | `str` | Raw unified diff text. Expected to contain one or more `@@ -A,B +C,D @@` lines (multi-line). Empty string returns `[]`. |

Edge cases on the input:
- Count field omitted (`@@ -10 +10 @@`) → treated as `count = 1` (per unified diff spec).
- Count of `0` (pure insertions like `@@ -10,0 +11,3 @@`, `@@ -0,0 +1,N @@`) → **skipped**, because no old line is actually affected.
- Trailing context after the closing `@@` (git's optional function-name suffix) → tolerated, because the regex isn't anchored to end-of-line.
- Non-hunk lines, `\ No newline at end of file`, binary diff markers, rename headers → silently ignored.
- `diff_output` not in unified diff format at all → returns `[]`.

## Return Value

`list[tuple[int, int]]` of **inclusive** old-side ranges.

Each tuple `(start, end)` means: old-file lines `start` through `end` (1-based, inclusive) are part of this hunk. Callers must treat the pair as a closed interval; e.g. overlap is checked with `hunk_end >= line_start and hunk_start <= line_end`.

Returned list is empty if there are no hunks, only pure-insertion hunks, or the input isn't a unified diff. Ordering follows the diff's natural ascending order (the function does not re-sort).

## Algorithm

1. Initialize `hunks = []`.
2. Compile-match `^@@ -(\d+)(?:,(\d+))? \+\d+(?:,\d+)? @@` with `re.MULTILINE`:
   - Group 1: OLD start line.
   - Group 2 (optional): OLD count. `None` when `,N` is omitted.
   - New-side fields are matched but **not captured** — they're consumed only so the regex anchors properly.
3. For each match:
   - `start = int(group(1))`.
   - `count = int(group(2)) if group(2) is not None else 1`.
   - If `count == 0`: `continue` (pure insertion into the old file — no old lines affected).
   - Append `(start, start + count - 1)`. This converts the `(start, count)` half-open-ish encoding into an inclusive `[start, end]` pair.
4. Return the collected list.

Step 3's `count - 1` is important: `@@ -10,3 @@` in unified diff means "starting at old line 10, three lines are old-side" — i.e. lines **10, 11, 12**, not 10–13.

## Side Effects

None. This is a pure function: no I/O, no mutation of arguments, no globals, no subprocess calls. It only reads `diff_output` and produces a new list.

## Error Handling

- **No exceptions are raised for malformed input.** The regex simply fails to match, and those lines are ignored.
- `int(...)` cannot raise here because the regex guarantees digits in both captured groups. The `if m.group(2) is not None` guard prevents `int(None)`.
- There is no defense against pathological or adversarial diff text beyond regex non-matching — the assumption is that input comes from `git diff`, not user-supplied data.

## Usage Patterns

The only in-file caller is `file_lines_changed_since`:

```python
result = subprocess.run(["git", "diff", "-U0", since_sha, "--", str(filepath.name)], ...)
for hunk_start, hunk_end in parse_diff_hunks(result.stdout):
    if hunk_end >= line_start and hunk_start <= line_end:
        return True
return False
```

Caller obligations:
- Must treat the returned pairs as **inclusive** closed intervals.
- Must remember the ranges reference the **OLD** side (i.e. the file at `since_sha`), which is exactly the coordinate system of the `pinned_lines` metadata (`"ls-le"`) set by `pin_lines`.
- Must handle the empty list (`[]`) as "no old lines changed" and fall through to whatever default check is appropriate.
- Because `-U0` is used, hunk ranges are already the minimal changed segments. If `-U3` were used, ranges would include context and could produce false positives when checking only a few pinned lines — hence the `-U0` coupling.

## Dependencies

- **`re`** — standard library, no compiled regex caching beyond Python's default (regex is recompiled each call; not a `re.compile` cache).
- Mutually, this function is coupled to:
  - `pin_lines` (produces `pinned_lines` as `"start-end"`)
  - `file_lines_changed_since` (the caller)
  - The git diff format produced by `git diff -U0 <sha> -- <file>`

## Assumptions Not Enforced by the Type System

- The string is a **unified diff** — not `diff -u` with different conventions, not `context` diff, not a JSON dump.
- Each `@@` header is on its own line (requires `MULTILINE`).
- The OLD-side field is the **first** coordinate pair after the dash; the new-side is discarded silently.
- `start` values in real unified diffs are `>= 1`; the code accepts `0` (which only legitimately occurs when `count == 0`, and those are skipped). A malformed `-0,N` with `N > 0` would yield `(0, N-1)` — not guarded against.
- Hunk headers may carry a trailing function/context suffix (`@@ ... @@ def foo()`), which the unanchored regex tolerates.
- Input order is ascending; the function does not sort. This is only implicit.
- Line numbers are within the file's actual line count at the pinned SHA; nothing validates this.

---

## Topics to Explore

- [function] `reasons/check_stale.py:file_lines_changed_since` — The sole caller; shows how inclusive ranges and `-U0` interact for the overlap check.
- [function] `reasons/check_stale.py:pin_lines` — The producer of `pinned_lines` metadata that these ranges are compared against; note its 1-based validation.
- [file] `tests/test_check_stale_issue25.py` — Regression tests likely covering the count=0 skip and boundary/overlap semantics.
- [general] `unified-diff-hunk-header-grammar` — The formal grammar of `@@ -A[,B] +C[,D] @@ [context]`, especially the "omitted count means 1" rule.
- [general] `git-diff-u0-semantics` — Why `-U0` produces pure-insertion/deletion hunks with count 0 and how they should be interpreted.

## Beliefs

- `parse-diff-hunks-returns-inclusive-ranges` — The end value is computed as `start + count - 1`, so tuples are inclusive closed intervals, not half-open.
- `parse-diff-hunks-skips-pure-insertions` — Hunks with old-side count `0` are dropped so that pure insertions (`@@ -N,0 +M,K @@`) never mark old lines as changed.
- `parse-diff-hunks-defaults-missing-count-to-one` — When the `,<count>` part of the old side is absent, the code substitutes `1`, matching the unified diff spec.
- `parse-diff-hunks-parses-old-side-only` — New-side start/count are matched by the regex but never captured or returned; only OLD-side `(start, end)` ranges are produced.
- `parse-diff-hunks-is-pure` — The function performs no I/O, spawns no subprocesses, and mutates no external state.
- `parse-diff-hunks-silently-ignores-nonmatching-lines` — Lines that don't match the hunk header regex (context, `\ No newline...`, renames) are skipped without error.
- `parse-diff-hunks-enables-git-aware-pinning` — Its output is what allows `check_stale` to skip content hashing for line-pinned nodes when only unrelated lines changed.
