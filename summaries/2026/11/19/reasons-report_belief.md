---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/report_belief.py

## Purpose

`reasons/report_belief.py` is the report-rendering layer for a single belief in the Truth Maintenance System. It does **not** compute truth, traverse the belief graph, fetch premises, or read source chunks itself. Instead, the caller gathers:

- the belief node,
- its explanation/justification chain,
- its root premises,
- source evidence chunks,

and this module turns that data into either:

1. a deterministic structured Markdown report, or
2. an LLM-generated narrative report when a `model` is supplied.

It owns the report format and the LLM prompt contract, not the truth-maintenance semantics.

---

## Key Components

### `REPORT_BELIEF_PROMPT`

A module-level prompt template used for LLM mode. It instructs the model to:

- summarize the belief and status,
- explain the justification chain,
- present source evidence for root premises,
- highlight premises that are OUT or lack evidence,
- avoid inventing data,
- omit a title because the wrapper adds one.

The `{data}` placeholder is filled by `_format_data_for_prompt(...)`.

### `_build_structured_report(...)`

Private deterministic renderer:

```python
_build_structured_report(
    node_id,
    node_detail,
    explain_steps,
    premises_data,
    source_chunks,
)
```

It returns a Markdown string with:

- `# Belief Report: \`node_id\``
- generation date and `node_detail["truth_value"]`
- the belief text as a blockquote
- optional belief `source`
- a `## Justification Chain` section
- a `## Root Premises (N)` section
- optional source evidence chunks rendered inside `<details><summary>...</summary>` blocks

Important formatting details:

- Truth markers are computed as:
  ```python
  marker = "+" if tv == "IN" else "-"
  ```
  So only the exact string `"IN"` gets `[+]`; `"OUT"`, `"UNKNOWN"`, missing values, etc. all get `[-]`.

- Each justification step may contain:
  - `node`
  - `truth_value`
  - `reason`
  - `antecedents`
  - `label`
  - `outlist`

- Each premise must contain `id`; other keys are optional:
  - `truth_value`
  - `text`
  - `source`

- `source_chunks` is expected to be a mapping from premise id to a list of chunk dicts with:
  - `filename`
  - `section`
  - `text`

If a premise id exists in `source_chunks` but maps to an empty list, the report emits:

```markdown
*No matching source chunks found.*
```

If the premise id simply is not a key in `source_chunks`, it emits no evidence section for that premise.

### `_format_data_for_prompt(...)`

Private formatter that converts the same gathered data into plain structured text for the LLM prompt. It includes:

- belief id, status, text, and source
- justification steps with node, truth value, reason, antecedents, and label
- root premises with id, truth value, text, source file, and source evidence

Notable difference from the structured renderer: `_format_data_for_prompt` does **not** include each step’s `outlist`, even though `_build_structured_report` does. Also, it does not emit a “no matching source chunks” note.

### `generate_report(...)`

```python
generate_report(data_text, model, timeout)
```

Builds the prompt via:

```python
prompt = REPORT_BELIEF_PROMPT.format(data=data_text)
```

Then lazily imports and calls:

```python
from .llm import invoke_model
return invoke_model(prompt, model=model, timeout=timeout)
```

The lazy import means deterministic/structured mode does not import the LLM layer at all.

### `report_belief(...)`

Public entry point:

```python
report_belief(
    node_id,
    node_detail,
    explain_steps,
    premises_data,
    source_chunks,
    model="",
    timeout=300,
)
```

Behavior:

- If `model` is truthy:
  - format data for the prompt,
  - call `generate_report(...)`,
  - wrap the LLM content with a title, generation date, and status line,
  - return Markdown.

- If `model` is empty/falsy:
  - call `_build_structured_report(...)`,
  - return deterministic Markdown.

`timeout` is only used in LLM mode.

---

## Patterns

- **Mode switch / strategy pattern**: `report_belief` selects deterministic rendering vs LLM narrative based on whether `model` is set.
- **Lazy import**: `invoke_model` is imported inside `generate_report`, keeping structured reporting independent of the LLM dependency at import time.
- **Pure rendering helpers**: `_build_structured_report` and `_format_data_for_prompt` perform no I/O and return strings from passed-in data.
- **Defensive dictionary access**: Optional fields are read with `.get(...)`, but required fields such as premise `id` use direct indexing.
- **Markdown-native rendering**: The structured report uses headings, blockquotes, fenced backticks, and `<details>` blocks directly rather than a Markdown library.
- **Prompt-template isolation**: The LLM instructions live in `REPORT_BELIEF_PROMPT`, separate from data formatting.

---

## Dependencies

### Depends on

- `datetime.date` from the standard library.
- `reasons.llm.invoke_model` lazily inside `generate_report`.

### Imported by

According to the repository context:

- `reasons/api.py`
- `reasons_lib/report_belief.py`
- `tests/test_report_belief.py`

`reasons/api.py` is likely the main orchestration caller that gathers report data and invokes `report_belief`. `reasons_lib/report_belief.py` appears to be a mirror or re-export in the `reasons_lib` namespace. Tests directly exercise this module.

---

## Flow

1. A caller such as `reasons/api.py` gathers:
   - `node_id`
   - `node_detail`
   - `explain_steps`
   - `premises_data`
   - `source_chunks`

2. It calls `report_belief(...)`.

3. If `model` is set:
   - `_format_data_for_prompt(...)` converts the structured data into LLM-readable text.
   - `generate_report(...)` inserts that text into `REPORT_BELIEF_PROMPT`.
   - `invoke_model(...)` generates the narrative.
   - `report_belief` prepends a title, date, and status line around the LLM output.

4. If `model` is empty:
   - `_build_structured_report(...)` walks through belief metadata, justification steps, root premises, and source chunks.
   - It appends Markdown lines and returns the joined string.

5. The returned value is always a Markdown string unless an exception occurs.

---

## Invariants

- `premises_data` entries must contain an `id` key; otherwise `_build_structured_report` raises `KeyError`.
- `source_chunks` must be a mapping supporting `.get(pid, [])`; passing `None` will raise `AttributeError`.
- Only `truth_value == "IN"` renders as `[+]`; all other values render as `[-]`.
- An empty `model` string selects structured rendering; any non-empty string selects LLM rendering.
- In LLM mode, the prompt explicitly tells the model not to include a title because `report_belief` adds one.
- The report does not validate or sanitize belief text, premise text, source filenames, or LLM output before inserting it into Markdown.
- The root premise count is simply `len(premises_data)`; it is not filtered by truth value.

---

## Error Handling

There is no explicit `try`/`except` in this module.

- Missing optional fields are tolerated via `.get(...)`.
- Missing required premise `id` raises `KeyError`.
- Passing `None` for `source_chunks` raises `AttributeError`.
- Errors from `invoke_model`, such as timeout or backend failure, propagate to the caller.
- If `model` is set and LLM generation fails, there is **no automatic fallback** to the structured report.
- The module does not validate LLM output; it wraps and returns whatever content `invoke_model` produces.

---

## Topics to Explore

- [file] `reasons/api.py` — See how callers gather `node_detail`, `explain_steps`, `premises_data`, and `source_chunks` before invoking this report renderer.
- [function] `reasons/llm.py:invoke_model` — Understand the LLM invocation contract, timeout behavior, and exception types propagated through `generate_report`.
- [file] `tests/test_report_belief.py` — Verify expected Markdown shape, truth markers, source evidence rendering, and LLM-mode wrapping behavior.
- [file] `reasons_lib/report_belief.py` — Confirm whether the `reasons_lib` namespace mirrors, re-exports, or duplicates this implementation.
- [general] `truth-maintenance-explanation-chain` — Understand the semantics of justification steps, antecedents, outlists, and root premises in this project.

---

## Beliefs

- `report-belief-returns-markdown` — `report_belief` always returns a Markdown string, using deterministic structured rendering when `model` is empty and LLM narrative rendering when `model` is non-empty.
- `structured-renderer-requires-premise-id` — `_build_structured_report` requires every `premises_data` element to contain an `id` key; missing it raises `KeyError`.
- `only-in-truth-value-gets-plus-marker` — The structured renderer maps only `truth_value == "IN"` to `[+]`; every other value, including `UNKNOWN`, maps to `[-]`.
- `source-chunks-must-be-a-mapping` — Both formatting helpers call `source_chunks.get(...)`, so callers must pass a mapping keyed by premise id; `None` raises `AttributeError`.
- `llm-failure-propagates-without-structured-fallback` — If `model` is set and `invoke_model` raises, `report_belief` propagates the exception instead of falling back to the structured report.
