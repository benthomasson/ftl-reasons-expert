---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/report_gated.py

# `reasons/report_gated.py`

## Purpose

This module owns **report generation for gated beliefs**. It takes already-collected data about:

- active blockers and the beliefs they gate,
- retracted premises that represent fixed defects,
- IN/OUT belief counts,
- optional per-blocker dependent-count details,

and produces a markdown report.

It does **not** query storage or discover blockers itself. The caller is responsible for assembling `gated_data`, `retracted_premises`, and `blocker_details`. This module only formats and/or synthesizes the report.

It supports two output modes:

1. **Deterministic structured markdown** when no model is supplied.
2. **LLM-generated narrative markdown** when `model` is truthy.

The public entry point is `report_gated()`.

---

## Key Components

### `REPORT_GATED_PROMPT`

A module-level prompt template used only in the LLM path.

It instructs the model to produce a professional markdown report with:

- a summary,
- a “Fixed Defects” section for retracted premises,
- an “Open Problems” section for active blockers,
- an “Impact Analysis” section,
- belief IDs in backticks,
- no invented information,
- no title, because the caller prepends one.

The template has one placeholder: `{data}`.

---

### `_build_structured_report(in_count, out_count, gated_data, retracted_premises, blocker_details)`

Builds the deterministic markdown report.

It emits:

- `# Gated Beliefs Report`
- a generated-date summary line using `date.today().isoformat()`
- optional `## Fixed Defects (Retracted Premises)` table
- optional `## Active Blockers` sections
- a fallback message if there are no blockers and no retracted premises
- always a `## Methodology` section

Important behavior:

- Retracted premises are sorted by `dependent_count` descending.
- The table row uses `` `{rp['id']}` — {text} ``.
- Retracted text is truncated to 80 characters and `|` is escaped as `\|`.
- `retract_reason` also escapes `|`, but is not truncated.
- Blockers are sorted by number of gated beliefs descending.
- Each blocker is numbered and rendered as `### {i}. `{bid}``.
- Gated beliefs inside a blocker are sorted by `g['id']`.
- Gated belief text is truncated to 100 characters.
- The summary uses `gated_data["blocker_count"]` and `gated_data["gated_count"]` directly; it does not recompute them from `blockers`.
- If blockers exist, it does not emit a “no blockers” fallback, even if there are retracted premises.
- If no blockers and no retracted premises exist, it emits `No blockers or retracted premises found.`

This path does **not** include an LLM-style “Impact Analysis” section. Its section names differ from the prompt’s requested sections: it uses “Active Blockers” where the prompt asks for “Open Problems.”

---

### `_format_data_for_prompt(in_count, out_count, gated_data, retracted_premises, blocker_details)`

Converts the same inputs into plain text for the LLM prompt.

It emits:

- total IN/OUT counts,
- blocker count and gated belief count,
- number of retracted premises,
- optional `### Fixed Defects` block,
- optional `### Active Blockers` block.

Differences from the structured report:

- It includes full belief text, without truncation.
- It does not build markdown tables.
- It includes each blocker’s total dependent count from `blocker_details.get(bid, {})`.
- Missing blocker detail entries default to `dependent_count = 0`.
- It sorts fixed defects by `dependent_count` descending.
- It sorts blockers by `len(bdata["gated"])` descending.
- It sorts gated beliefs by `g["id"]`.

The output is inserted into `REPORT_GATED_PROMPT.format(data=data_text)`.

---

### `generate_report(data_text, model, timeout)`

A thin LLM wrapper.

It:

1. Lazily imports `invoke_model` from `.llm`.
2. Formats `REPORT_GATED_PROMPT` with `data_text`.
3. Calls `invoke_model(prompt, model=model, timeout=timeout)`.
4. Returns the model result.

The lazy import means importing `report_gated.py` does not require `.llm` to be imported at module load time.

---

### `report_gated(...)`

The public entry point:

```python
report_gated(
    gated_data,
    retracted_premises,
    blocker_details,
    in_count,
    out_count,
    model="",
    timeout=300,
)
```

Behavior:

- If `model` is truthy:
  - calls `_format_data_for_prompt(...)`,
  - calls `generate_report(...)`,
  - prepends `# Gated Beliefs Report`,
  - prepends the generated-date summary line,
  - appends the LLM content,
  - returns markdown.

- If `model` is falsy, typically `""` or `None`:
  - calls `_build_structured_report(...)`,
  - returns deterministic markdown.

The `timeout` argument is only used in the LLM path. It is ignored by the structured path.

Both paths return a markdown string beginning with the top-level report title.

---

## Patterns

### Strategy selection by `model`

The main behavioral switch is:

```python
if model:
    ...
return _build_structured_report(...)
```

A non-empty model string selects the LLM strategy. An empty string or `None` selects the deterministic structured strategy.

There is no separate strategy class; the code uses simple function dispatch.

### Lazy optional dependency

`generate_report()` imports `invoke_model` inside the function rather than at module import time:

```python
from .llm import invoke_model
```

This avoids importing the LLM integration unless the LLM path is actually used.

### Deterministic ordering

Both report paths sort data:

- retracted premises by `dependent_count` descending,
- blockers by number of gated beliefs descending,
- gated beliefs by ID ascending.

This keeps report output stable for tests and human review.

### Defensive defaults, but no full validation

The code uses `.get()` for optional mappings:

```python
blockers = gated_data.get("blockers", {})
detail = blocker_details.get(bid, {})
dep_count = detail.get("dependent_count", 0)
```

However, required keys such as `gated_data["blocker_count"]`, `bdata["text"]`, and `rp["retract_reason"]` are accessed directly. Malformed input therefore raises normal Python exceptions.

### Minimal markdown escaping

Only the retracted-premise table escapes pipes:

```python
text = rp["text"][:80].replace("|", "\\|")
reason = rp["retract_reason"].replace("|", "\\|")
```

Other text fields, including blocker text and gated belief text, are inserted without markdown escaping or newline sanitization.

---

## Dependencies

### Depends on

- `datetime.date`

  Used for the generated-date line.

- `.llm.invoke_model`

  Imported lazily inside `generate_report()`. Used only when `model` is truthy.

- Expected input data shape

  The module assumes callers pass dictionaries/lists matching the shapes used by the report code:

  - `gated_data`:
    - `blocker_count`
    - `gated_count`
    - optional `blockers`: mapping `blocker_id -> {"text": str, "gated": [{"id": str, "text": str}, ...]}`

  - `retracted_premises`: list of:
    - `id`
    - `text`
    - `retract_reason`
    - `dependent_count`

  - `blocker_details`: mapping `blocker_id -> {"dependent_count": int}`

### Imported by

Per the repository context:

- `reasons/api.py`
- `reasons_lib/report_gated.py`
- `tests/test_report_gated.py`

So this module sits below the public API layer and is directly tested. The `reasons_lib` mirror/import graph also depends on it, so changes here may need mirror review.

---

## Flow

### Structured path

`report_gated(model="")`
→ `_build_structured_report(...)`
→ build title
→ build summary line
→ optionally build fixed-defects table
→ optionally build active-blockers sections
→ optionally add no-data fallback
→ always add methodology
→ return joined markdown.

### LLM path

`report_gated(model="some-model")`
→ `_format_data_for_prompt(...)`
→ `generate_report(data_text, model, timeout)`
→ lazy import `invoke_model`
→ format `REPORT_GATED_PROMPT`
→ call LLM
→ prepend title and summary
→ append LLM content
→ return joined markdown.

---

## Invariants

- `report_gated()` always returns a markdown string starting with `# Gated Beliefs Report`.
- A truthy `model` selects the LLM path; a falsy `model` selects the structured path.
- The generated-date summary line is present in both paths.
- The LLM path prepends the title and summary itself; the prompt explicitly tells the model not to include a title.
- Structured output truncates retracted text to 80 characters and gated belief text to 100 characters.
- Prompt/LLM input formatting uses full text, not truncated text.
- Structured output escapes `|` only in retracted-premise table fields.
- Summary counts `blocker_count` and `gated_count` are trusted from `gated_data`; they are not recomputed or cross-validated against `blockers`.
- Blockers are sorted by number of gated beliefs descending.
- Gated beliefs are sorted by ID ascending.
- Retracted premises are sorted by dependent count descending.
- Missing `blocker_details[bid]` is treated as `dependent_count = 0`.
- The structured report always includes a Methodology section.
- The LLM-generated report does not automatically include the structured report’s Methodology section.

---

## Error Handling

There is no `try/except` in this module.

Consequences:

- Missing required keys raise `KeyError`.
- Missing or `None` values where strings/lists are expected can raise `TypeError` or `AttributeError`.
- If `invoke_model()` raises, the exception propagates to the caller.
- There is no fallback from the LLM path to the structured path on LLM failure.
- If `model` is falsy, `.llm` is never imported, so LLM-related import errors are avoided.
- Markdown escaping is minimal; malformed text containing newlines or unescaped markdown can break formatting in the generated report.
- `timeout` is passed through unchanged in the LLM path; the module does not validate it.

---

## Topics to Explore

- [file] `reasons/api.py` — Shows how `report_gated()` is called and how `gated_data`, `retracted_premises`, and `blocker_details` are assembled.
- [file] `tests/test_report_gated.py` — Captures expected output, sorting, truncation, and LLM/non-LLM path behavior.
- [function] `reasons/llm.py:invoke_model` — Defines the LLM invocation contract, timeout behavior, and error semantics relied on by `generate_report()`.
- [file] `reasons_lib/report_gated.py` — The mirror/import-layer counterpart that may need to stay in sync with this module.
- [general] `gated-belief-data-shape` — Trace how `list-gated` and `show` data are shaped upstream so the required keys and counts are correct.

## Beliefs

- `report-gated-selects-llm-path-when-model-is-truthy` — If `model` is non-empty, `report_gated()` uses `_format_data_for_prompt()` and `generate_report()`; otherwise it uses `_build_structured_report()`.
- `llm-path-prepends-title-and-summary` — In the LLM path, `report_gated()` prepends `# Gated Beliefs Report` and the generated-date summary before the LLM content.
- `structured-report-truncates-retracted-and-gated-text` — The structured path truncates retracted premise text to 80 characters and gated belief text to 100 characters; the prompt path does not truncate.
- `structured-report-escapes-pipes-only-in-retracted-table` — Only retracted premise text and `retract_reason` replace `|` with `\|`; blocker and gated belief text are not escaped.
- `report-gated-has-no-llm-fallback` — If `invoke_model()` raises, the exception propagates and the code does not fall back to the structured report.
