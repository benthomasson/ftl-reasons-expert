---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: The formal grammar of `@@ -A[,B] +C[,D] @@ [context]`, especially the "omitted count means 1" rule.

## Formal grammar of `@@ -A[,B] +C[,D] @@ [context]`

This is the standard unified-diff hunk header. In general form:

```text
@@ -A[,B] +C[,D] @@ [context]
```

where:

- `A` = old-file starting line number
- `B` = old-file line count, optional
- `C` = new-file starting line number
- `D` = new-file line count, optional
- `[context]` = optional section heading / function context after the closing `@@`

A compact EBNF form is:

```ebnf
hunk-header = "@@ -" old-range " +" new-range " @@" [ context ]
old-range   = old-start [ "," old-count ]
new-range   = new-start [ "," new-count ]
old-start   = digit+
new-start   = digit+
old-count   = digit+
new-count   = digit+
context     = any text to end of line
```

The key rule is: **if a count is omitted, it defaults to `1`.** So:

- `@@ -42 +42 @@` means old range starts at line 42 with count 1, i.e. line 42 only.
- `@@ -42 +42,2 @@` means old range is line 42 only, new range is lines 42–43.
- `@@ -89,5 +89,7 @@` means old range is lines 89–93 and new range is lines 89–95.

The implementation you observed is narrower than the full unified-diff grammar. In `reasons/check_stale.py:257`, the parser uses:

```python
r'^@@ -(\d+)(?:,(\d+))? \+\d+(?:,\d+)? @@'
```

That regex:

- captures only the **old start** (`A`) and optional **old count** (`B`)
- matches the new side `+C[,D]` syntactically but does **not capture** `C` or `D`
- allows the old count and new count to be omitted
- does not capture or validate the optional `[context]` after the closing `@@`
- requires the header to appear at the beginning of a line because of `^` with `re.MULTILINE`

The “omitted count means 1” rule is implemented explicitly at `reasons/check_stale.py:259`:

```python
count = int(m.group(2)) if m.group(2) is not None else 1
```

So when the old-count capture group is `None`, which happens for a header like `@@ -42 +42,2 @@`, the parser substitutes `1`. Then at `reasons/check_stale.py:262` it converts the header into an inclusive old-file line range:

```python
hunks.append((start, start + count - 1))
```

Examples from the tests confirm this:

- `tests/test_pin_sources.py:291-292`:  
  `@@ -89,5 +89,7 @@ def some_function():` → `(89, 93)`
- `tests/test_pin_sources.py:296-300`:  
  `@@ -10,3 +10,4 @@ ...` and `@@ -50,2 +51,2 @@ ...` → `(10, 12)`, `(50, 51)`
- `tests/test_pin_sources.py:303-304`:  
  `@@ -42 +42,2 @@` → `(42, 42)` because the omitted old count defaults to `1`
- `tests/test_pin_sources.py:307-308`:  
  `@@ -89,0 +90,3 @@` → `[]`, showing that zero-count old hunks are excluded rather than producing an inverted range

The zero-count case is important. In unified diff syntax, a count of `0` means “no lines from that side”, commonly used for pure insertions or deletions. The test at `tests/test_pin_sources.py:306-308` asserts that `@@ -89,0 +90,3 @@` produces no old-file range. The observations do not show the full guard between `reasons/check_stale.py:259` and `reasons/check_stale.py:262`, but the test establishes the behavior: zero old count is skipped.

The optional `[context]` part is ignored by this parser. Because the regex ends at the second `@@` and is not anchored with `$`, a header like:

```text
@@ -89,5 +89,7 @@ def some_function():
```

still matches; the `def some_function():` portion is simply not captured. That context is useful for human readability and some diff tools, but `parse_diff_hunks` only needs the old-file line range for the stale-checking logic.

In short, for this codebase:

- The full header shape is recognized as `@@ -A[,B] +C[,D] @@ [context]`.
- The parser only uses `A` and `B`.
- `B` omitted means `B = 1`.
- The result is an inclusive old-file range `(A, A + B - 1)`.
- `B = 0` is filtered out.
- `C`, `D`, and `[context]` are matched or tolerated but not used.

## Topics to Explore

- [file] `reasons/check_stale.py` — Read the full `parse_diff_hunks` implementation around lines 251–262 to see the exact zero-count guard and surrounding logic.
- [function] `reasons/check_stale.py:parse_diff_hunks` — Understand how hunk headers are converted into old-file inclusive ranges and consumed by stale checking.
- [file] `tests/test_pin_sources.py` — Review `TestParseDiffHunks` for concrete examples of omitted counts, multiple hunks, zero counts, and malformed input.
- [general] `unified-diff-hunk-header-spec` — Compare this narrow parser against the broader unified-diff spec, especially new-side counts and context headings.
- [general] `git-diff-zero-count-semantics` — Explore how pure insertions/deletions use `-A,0` or `+C,0` and why zero-count hunks are handled specially.

## Beliefs

- `parse-diff-hunks-defaults-omitted-old-count-to-one` — In `reasons/check_stale.py`, an omitted old count is treated as `1` at line 259.
- `parse-diff-hunks-produces-inclusive-old-line-ranges` — `parse_diff_hunks` appends `(start, start + count - 1)`, producing inclusive old-file line ranges.
- `parse-diff-hunks-ignores-new-side-ranges` — The regex matches `+C[,D]` but does not capture or use the new-file start or count.
- `parse-diff-hunks-excludes-zero-old-count` — Tests assert that `@@ -89,0 +90,3 @@` yields no hunk range.
- `parse-diff-hunks-ignores-context-after-closing-double-at` — The regex stops at the closing `@@`, so optional context headings are not captured or validated.
