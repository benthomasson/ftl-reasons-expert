---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: _build_structured_report in reasons/report_gated.py

## Purpose

`_build_structured_report` builds a deterministic Markdown “Gated Beliefs Report” without invoking an LLM. It is the fallback path used by `report_gated` when no model is configured, or used directly by tests/internal callers.

The report summarizes:

- IN/OUT belief counts
- active blockers and which beliefs they gate
- retracted premises as “fixed defects”
- a short methodology note

It exists so the CLI can always produce a useful report even when LLM generation is unavailable or undesired.

## Contract

### Preconditions

The function assumes a fairly specific data shape, though none of it is enforced by type annotations:

- `gated_data` is a mapping containing at least:
  - `"blocker_count"` — integer-like count
  - `"gated_count"` — integer-like count
  - optionally `"blockers"` — mapping of blocker ID to blocker data
- Each blocker value, if present, is expected to contain:
  - `"text"` — string
  - `"gated"` — list of dicts with `"id"` and `"text"`
- `retracted_premises` is an iterable of dicts, each expected to contain:
  - `"id"`
  - `"text"`
  - `"retract_reason"`
  - `"dependent_count"`
- `blocker_details` is a mapping of blocker ID to detail dicts, typically containing `"dependent_count"`. Missing blocker IDs are tolerated.

### Postconditions

- Returns a Markdown string.
- Starts with `# Gated Beliefs Report`.
- Always includes the generated-date/count summary line.
- Includes “Fixed Defects” section only when `retracted_premises` is truthy.
- Includes “Active Blockers” section only when `gated_data["blockers"]` is truthy.
- Includes “No blockers or retracted premises found.” only when both `blockers` and `retracted_premises` are falsy.
- Always includes a final “Methodology” section.
- Does not mutate the input containers or dicts.
- Returns a string ending with a newline, because the last appended line is an empty string before `"\n".join(lines)`.

### Important invariants / assumptions

- `blocker_count` and `gated_count` are taken directly from `gated_data`; they are **not recomputed** from `blockers`.
- Retracted premises are sorted by `dependent_count` descending.
- Active blockers are sorted by number of gated beliefs descending.
- Gated beliefs within each blocker are sorted by `id` ascending.
- Markdown pipe escaping is only applied to `retracted_premises` text and retract reason in the table. Blocker text and gated belief text are not escaped.
- IDs and texts are assumed to be strings. Missing keys or wrong types will generally raise exceptions.

## Parameters

### `in_count`

Expected integer-like count of IN beliefs. It is inserted directly into the summary line.

### `out_count`

Expected integer-like count of OUT beliefs. Inserted directly into the summary line.

### `gated_data`

A mapping. Required keys:

- `"blocker_count"` — used in the summary line
- `"gated_count"` — used in the summary line

Optional key:

- `"blockers"` — mapping of blocker ID to blocker record

The code does:

```python
blockers = gated_data.get("blockers", {})
```

so a missing `"blockers"` key is treated as no blockers. However, `"blocker_count"` and `"gated_count"` are accessed with `[]`, so missing them raises `KeyError`.

Each blocker record is expected to have:

```python
{
    "text": "...",
    "gated": [
        {"id": "...", "text": "..."},
        ...
    ],
}
```

### `retracted_premises`

An iterable of retracted-premise records. It is checked for truthiness and then sorted. Each record is expected to contain:

```python
{
    "id": "...",
    "text": "...",
    "retract_reason": "...",
    "dependent_count": int,
}
```

If `dependent_count` is `0`, the report renders `"no dependents"`; otherwise it renders `"N dependent(s)"`.

Edge case: if a generator is passed, truthiness is not list emptiness, so an empty generator would still enter the section and produce an empty table.

### `blocker_details`

A mapping of blocker ID to additional detail. The only field used is:

```python
detail.get("dependent_count", 0)
```

If a blocker ID is missing from `blocker_details`, the dependent count defaults to `0`. If `blocker_details` itself is `None` and there are blockers, `.get` will fail with `AttributeError`.

## Return Value

Returns a single Markdown string.

The caller is responsible for writing it to a file, printing it, or returning it through the CLI. There is no error-return object and no status code. Any data-shape problems raise exceptions rather than being captured in the returned string.

The final string always contains:

- title
- generated summary
- optional fixed-defects table
- optional active-blockers section
- optional no-issues message
- methodology section

## Algorithm

1. Create an empty list of lines.
2. Append `# Gated Beliefs Report` and a blank line.
3. Read `blocker_count` and `gated_count` from `gated_data`.
4. Get today’s date via `date.today().isoformat()`.
5. Append the italicized summary line with IN/OUT counts and blocker/gated counts.
6. If `retracted_premises` is truthy:
   - Append the “Fixed Defects” heading.
   - Append a Markdown table header.
   - Sort retracted premises by `dependent_count` descending.
   - For each one:
     - compute impact text
     - truncate text to 80 characters
     - escape `|` characters in text and retract reason
     - append a table row with the ID in backticks.
7. Read `blockers = gated_data.get("blockers", {})`.
8. If `blockers` is truthy:
   - Append “Active Blockers” heading.
   - Sort blocker items by `len(bdata["gated"])` descending.
   - Enumerate blockers starting at `1`.
   - For each blocker:
     - look up `blocker_details.get(bid, {})`
     - get `dependent_count`, defaulting to `0`
     - append numbered heading with blocker ID
     - append blocker text
     - append gate/dependency summary
     - sort gated beliefs by `id` ascending
     - append each gated belief as a bullet with text truncated to 100 characters.
9. Otherwise, if there are no blockers and no retracted premises:
   - Append `No blockers or retracted premises found.`
10. Always append the “Methodology” section.
11. Join all lines with `"\n"` and return the result.

## Side Effects

Minimal:

- Calls `date.today()`, which reads the current local system date.
- Allocates new lists and strings.
- Does not write files, print, make network calls, or mutate input arguments.
- Sorting produces new lists; the original input ordering is not changed.

## Error Handling

There is no explicit error handling and no `try/except`.

Common exceptions:

- `KeyError`
  - missing `gated_data["blocker_count"]`
  - missing `gated_data["gated_count"]`
  - missing `rp["dependent_count"]`, `rp["text"]`, etc.
  - missing `bdata["text"]` or `bdata["gated"]`
  - missing `g["id"]` or `g["text"]`
- `TypeError`
  - sorting non-numeric `dependent_count`
  - slicing non-string text
  - joining non-string entries if malformed data enters `lines`
- `AttributeError`
  - calling `.replace` on a non-string / `None`
  - calling `.get` on `None` `blocker_details` when blockers exist
  - calling `.items()` on a non-mapping `blockers`
- `IndexError` is not expected here.
- No errors are swallowed.

The function does gracefully tolerate:

- missing `"blockers"` key
- missing individual blocker entries in `blocker_details`
- empty `retracted_premises`
- empty/missing blockers

But it does not validate consistency between `blocker_count`, `gated_count`, and the actual `blockers` contents.

## Usage Patterns

Typically called from `report_gated`:

```python
return _build_structured_report(
    in_count, out_count, gated_data, retracted_premises,
    blocker_details,
)
```

It is the non-LLM branch. When `model` is provided, `report_gated` instead formats data with `_format_data_for_prompt` and calls the LLM.

Caller obligations:

- Provide all required dict keys.
- Ensure strings are strings, counts are sortable numeric values, and blocker/gated structures match the expected shape.
- Ensure `blocker_details` is a mapping if blockers are present.
- Handle the returned string as report content.

## Dependencies

Internal:

- No other functions from `reasons.report_gated` are called.
- The shape of `gated_data`, `retracted_premises`, and `blocker_details` is coupled to whatever produces them elsewhere, likely CLI commands such as `reasons list-gated` and `reasons show`.

External:

- `datetime.date` imported at module level.
- Python built-ins: `sorted`, `enumerate`, `len`, string slicing/replace/join.

## Topics to Explore

- [file] `reasons/report_gated.py` — Full module showing both structured and LLM report generation paths.
- [function] `reasons/report_gated.py:_format_data_for_prompt` — How the same input data is serialized for the LLM branch.
- [function] `reasons/report_gated.py:report_gated` — Public entry point that decides whether to use this structured builder or an LLM.
- [file] `tests/test_report_gated.py` — Expected Markdown output and edge-case coverage for this report builder.
- [general] `gated-report-data-contract` — The actual producer-side shapes for `gated_data`, `retracted_premises`, and `blocker_details`.

## Beliefs

- `structured-report-is-llm-free` — `_build_structured_report` produces Markdown by string assembly only and never calls an LLM.
- `structured-report-requires-gated-data-count-keys` — It unconditionally reads `gated_data["blocker_count"]` and `gated_data["gated_count"]`, so missing keys raise `KeyError`.
- `structured-report-sorts-blockers-by-gated-count-descending` — Active blockers are ordered by descending `len(bdata["gated"])`, with ties preserving original mapping order due to stable sorting.
- `structured-report-defaults-missing-blocker-detail-count-to-zero` — A blocker missing from `blocker_details` gets `dependent_count` `0`, while missing `text` or `gated` in blocker data raises.
- `structured-report-does-not-recompute-counts` — The summary blocker/gated counts come directly from `gated_data` and are not validated against the actual `blockers` mapping.
