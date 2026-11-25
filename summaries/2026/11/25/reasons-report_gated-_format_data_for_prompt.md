---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: _format_data_for_prompt in reasons/report_gated.py

## `_format_data_for_prompt` — Explanation

### 1. Purpose

This helper converts the three raw data structures gathered by the report-gating logic (counts, retracted premises, active blockers) into a single, structured plaintext block. That block is later substituted into `REPORT_GATED_PROMPT`'s `{data}` placeholder and handed to an LLM. It is the "data serialization" step of the LLM-backed report path: it exists so the prompt template can stay generic while the actual network facts are rendered in a stable, readable layout (headings, `Key: value` lines, bullet-style nested items) that language models handle well.

It is used **only** when `model` is non-empty in `report_gated`; the no-LLM path uses `_build_structured_report` instead, which produces the final markdown output rather than an intermediate data digest.

### 2. Contract

- **Preconditions**
  - `gated_data` is a `dict` containing at least keys `blocker_count` and `gated_count`; optionally `blockers`.
  - If `gated_data["blockers"]` exists, it maps blocker ID → `dict` with keys `text` and `gated` (an iterable of dicts each having `id` and `text`).
  - `retracted_premises` is an iterable of `dict`s, each with keys `id`, `text`, `retract_reason`, `dependent_count`.
  - `blocker_details` is a mapping from blocker ID → `dict` that may contain `dependent_count`.
- **Postconditions**
  - Returns a single `str` with `\n`-joined lines; the string is safe to pass through `str.format(data=...)` (no literal braces are introduced — see caveat below).
  - Inputs are not mutated.
- **Invariants**
  - Header lines (totals + retracted count) are always present, followed by a blank line.
  - The "Fixed Defects" section appears only if `retracted_premises` is truthy.
  - The "Active Blockers" section appears only if the resolved `blockers` mapping is truthy.
  - Each blocking group ends with a trailing blank line so sections are visually separated when concatenated.

### 3. Parameters

| Parameter | Expected type | Meaning | Edge cases |
|---|---|---|---|
| `in_count` | `int` | Count of currently IN beliefs. | Rendered verbatim via `f"…{in_count}…"`. `None`/str would still stringify, but semantically wrong. |
| `out_count` | `int` | Count of OUT beliefs. | Same as above. |
| `gated_data` | `dict` | Aggregate blocker stats plus optional `"blockers"` map. | Missing `blocker_count`/`gated_count` → `KeyError`. Missing `"blockers"` → treated as `{}` via `.get`. |
| `retracted_premises` | `list[dict]` | Fixed defects (previously retracted premises). | Empty list suppresses the section; entries must contain `id`, `text`, `retract_reason`, `dependent_count`. |
| `blocker_details` | `dict[str, dict]` | Extra impact info per blocker ID. | Missing entry is tolerated (`blocker_details.get(bid, {})` → `{}` and `dependent_count` defaults to 0). |

### 4. Return Value

A single `str`, the digest that the caller injects into `REPORT_GATED_PROMPT.format(data=data_text)`. The caller (`report_gated`) then wraps the LLM's response with a top-level title and a generated-date summary line; the returned value itself contains **no** `#` H1 title.

The caller must ensure the returned string contains no unescaped `{` or `}` characters, because it is fed through `.format(data=...)`. Since values from `text`, `retract_reason`, etc., are inserted with f-strings (not escaped), belief text containing braces would break prompt formatting. Nothing in this function guards against that.

### 5. Algorithm

1. Initialize `lines = []`.
2. Append summary lines: `"Total beliefs: X IN, Y OUT"`, then `"Blockers: B gating G belief(s)"`, then `"Retracted premises (fixed defects): N"`, then `""`.
3. If `retracted_premises` is truthy:
   - Emit `### Fixed Defects` heading + blank line.
   - Sort premises **descending by `dependent_count`** (`key=lambda r: -r["dependent_count"]`) so the highest-impact defects appear first.
   - For each premise, emit four lines (`ID:`, `Text:`, `Retract reason:`, `Dependent count:`) followed by a blank line.
4. Resolve `blockers = gated_data.get("blockers", {})`.
5. If `blockers` is truthy:
   - Emit `### Active Blockers` heading + blank line.
   - Sort blockers **descending by `len(bdata["gated"])`** — the blocker gating the most beliefs appears first.
   - For each blocker: look up detail (default `{}`), emit `Blocker ID:`, `Blocker text:`, `Total dependents: N` (defaulting to 0), then a `Gates N belief(s):` line, then a bullet list of gated beliefs **sorted ascending by `id`** (`key=lambda g: g["id"]`), then a blank line.
6. Return `"\n".join(lines)`.

### 6. Side Effects

None. The function is pure with respect to object state: it builds a local list of strings, calls `sorted` on inputs (which returns new lists, not in-place), and performs no I/O, logging, or global mutation. It does **not** call the LLM — that happens in `generate_report` after this returns.

### 7. Error Handling

No explicit error handling. Exceptions that can propagate:

- `KeyError` if `gated_data["blocker_count"]` or `gated_data["gated_count"]` are missing (direct subscription), or if any `rp["id"]/["text"]/["retract_reason"]/["dependent_count"]` or `bdata["text"]/["gated"]` or `g["id"]/["text"]` are missing.
- `TypeError` if `retracted_premises` or `bdata["gated"]` is not iterable, or if `dependent_count` is not numeric (unary `-` in sort key).
- `AttributeError` if `blockers` isn't a dict (no `.items()`).

Defensive defaults used: `gated_data.get("blockers", {})`, `blocker_details.get(bid, {})`, `detail.get("dependent_count", 0)`. These are the only "tolerated" missing fields. Note asymmetry with `_build_structured_report`, which uses the same `.get` patterns — so both rendering paths share the same shape contract.

### 8. Usage Patterns

Called exactly once, from `report_gated` (in `reasons/report_gated.py`), and only on the LLM branch:

```python
if model:
    data_text = _format_data_for_prompt(in_count, out_count, gated_data,
                                        retracted_premises, blocker_details)
    content = generate_report(data_text, model, timeout)
    ...
```

**Caller obligations:**
- Do not call when the structured path is desired — that is the `_build_structured_report` route.
- Ensure belief `text`/`retract_reason` strings contain no `{`/`}` characters, since the output flows through `str.format` via `REPORT_GATED_PROMPT.format(data=...)`.
- Ensure all dicts conform to the shapes above; the function intentionally does not validate callers.

It is an underscore-prefixed "private" helper for the module.

### 9. Dependencies

- **Standard library:** none imported inside the function; only builtins (`sorted`, `len`, `f-strings`, `dict.get`).
- **Module-internal:** none. It is intentionally decoupled from `REPORT_GATED_PROMPT` (the caller wires them together) and from `generate_report`/`invoke_model` (the prompt adapter lives in `reasons.llm`, imported only inside `generate_report`).
- **Implicit data source:** trusts the shape produced by whatever gathers `gated_data`, `retracted_premises`, and `blocker_details` upstream (presumably `reasons list-gated` / `reasons show` CLI paths — see nearby module docstring).

### Assumptions not enforced by the type system

- Every dict in `retracted_premises` has all four keys and `dependent_count` is numeric (`-r["dependent_count"]`).
- `bdata["gated"]` is iterable; sorting `g["id"]` assumes IDs are orderable (typically `str`).
- `gated_data["blockers"]`, when present, is a `dict`, and its values are per-blocker dicts — not lists or `None`.
- No string value contains `{`/`}`.
- No belief IDs collide in `blocker_details` (dict keyed by ID); a colliding ID would silently attribute one blocker's `dependent_count` to another — not detectable here.

---

## Topics to Explore

- [function] `reasons/report_gated.py:report_gated` — The only caller; shows how `data_text` becomes the final LLM-backed report via `REPORT_GATED_PROMPT`.
- [function] `reasons/report_gated.py:_build_structured_report` — The non-LLM counterpart, useful for comparing formatting/impact presentation with the digest produced here.
- [file] `reasons/report_gated.py` — Full module including the prompt template, which dictates how this function's output will be consumed by the model.
- [file] `tests/test_report_gated.py` — Confirms the expected shapes and any edge cases (empty lists, missing `blocker_details`) currently covered.
- [general] `gated-belief-data-shapes` — Trace where `gated_data`, `retracted_premises`, and `blocker_details` are produced (CLI/`reasons list-gated`) to understand why certain keys are `.get`-defaulted while others are hard-indexed.

## Beliefs

- `format-data-for-prompt-produces-prompt-data-block` — Returns a newline-joined string consumed as `{data}` in `REPORT_GATED_PROMPT.format(...)` by `generate_report`.
- `format-data-for-prompt-requires-gated-data-counts` — Directly indexes `gated_data["blocker_count"]` and `gated_data["gated_count"]`, raising `KeyError` if either is absent.
- `format-data-for-prompt-defaults-missing-blocker-details` — Missing `blocker_details[bid]` falls back to `{}` and `dependent_count` defaults to `0`.
- `format-data-for-prompt-sorts-retracted-by-dependent-count-desc` — Retracted premises are ordered by descending `dependent_count`.
- `format-data-for-prompt-sorts-blockers-by-gated-count-desc` — Blockers are ordered by descending `len(bdata["gated"])`, and gated beliefs sorted ascending by `id`.
- `format-data-for-prompt-is-pure` — The function performs no I/O and does not mutate its arguments; only local `lines` and `sorted` results are produced.
