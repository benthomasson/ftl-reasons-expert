---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: report_gated in reasons/report_gated.py

## Purpose

`report_gated` is the public entry point for rendering a **gated-beliefs / problems report** as Markdown. It does not gather the underlying belief-network data; the caller supplies already-collected data, and this function decides how to render it:

- If `model` is falsy, it delegates to `_build_structured_report` for a deterministic, hand-built Markdown report.
- If `model` is truthy, it formats the data for an LLM prompt, calls `generate_report`, and wraps the LLM output with a title and generated-date summary.

It exists so callers have one stable report API with an optional narrative/LLM mode, while preserving a deterministic fallback path.

## Contract

Preconditions, none of which are enforced by type hints:

- `gated_data` must be a mapping containing:
  - `"blocker_count"`
  - `"gated_count"`
  - optionally `"blockers"`, a mapping of blocker ID to a dict with `"text"` and `"gated"` list entries.
- Each `gated` entry must contain `"id"` and `"text"`.
- `retracted_premises` is expected to be a list/sequence of dicts with:
  - `"id"`
  - `"text"`
  - `"retract_reason"`
  - `"dependent_count"`
- `blocker_details` is expected to be a mapping of blocker ID to a dict; only `"dependent_count"` is read, defaulting to `0`.
- `in_count` and `out_count` are expected to be numeric counts.
- `model` is used as a truthiness switch and as the model name passed to the LLM layer.
- `timeout` is passed to the LLM call and is ignored in structured mode.

Postconditions:

- Always returns a Markdown string beginning with `# Gated Beliefs Report`.
- Includes a generated-date summary line with `in_count`, `out_count`, `blocker_count`, and `gated_count`.
- In structured mode, includes deterministic sections such as `## Fixed Defects (Retracted Premises)`, `## Active Blockers`, and `## Methodology`.
- In LLM mode, appends the raw LLM-generated content after the title/summary header.

Important invariant/assumption: the summary counts come directly from `gated_data`, not from recomputing the rendered sections. If those counts disagree with `gated_data["blockers"]` or `retracted_premises`, the report can be internally inconsistent.

## Parameters

- `gated_data: dict`
  - Required keys: `"blocker_count"`, `"gated_count"`.
  - Optional key: `"blockers"`; if missing, treated as `{}`.
  - Each blocker value is expected to have `"text"` and `"gated"`.
  - Missing required keys raise `KeyError`.

- `retracted_premises: list[dict]`
  - Used to render fixed defects.
  - Expected fields: `id`, `text`, `retract_reason`, `dependent_count`.
  - Structured mode truncates `text` to 80 chars and escapes `|` in `text` and `retract_reason`.
  - LLM mode also calls `len(retracted_premises)`, so a generator would be problematic even though structured mode could sort one.

- `blocker_details: dict`
  - Maps blocker ID to detail data.
  - Only `dependent_count` is used, with default `0`.
  - If a key exists but maps to `None` instead of a dict, `.get` will raise `AttributeError`.

- `in_count`, `out_count`
  - Counts of IN/OUT beliefs.
  - Used only in the summary line and prompt data.

- `model: str`
  - Empty string / falsy value => structured mode.
  - Any truthy value, including whitespace, => LLM mode.
  - Not validated against available model names.

- `timeout: int`
  - Passed through to `generate_report` and then `invoke_model`.
  - Ignored entirely in structured mode.

## Return Value

Returns a Markdown string.

In both modes it starts with:

```markdown
# Gated Beliefs Report
```

and includes a line like:

```markdown
*Generated YYYY-MM-DD — N IN / M OUT beliefs, B active blocker(s) gating G belief(s)*
```

In LLM mode, the returned string contains that header, then the raw LLM content, then a trailing newline via the final empty `lines` entry. In structured mode, `_build_structured_report` produces the full deterministic document.

The caller must handle exceptions; this function never returns an error object or fallback report.

## Algorithm

1. Evaluate `if model:`.
2. If `model` is truthy:
   - Call `_format_data_for_prompt(...)` to build a plain-text representation of the network data.
   - Call `generate_report(data_text, model, timeout)`.
   - `generate_report` formats `REPORT_GATED_PROMPT` with `data_text` and calls `invoke_model(...)`.
   - Build the outer Markdown:
     - title
     - generated-date summary line using `date.today().isoformat()`
     - blank line
     - raw LLM content
     - trailing blank line
   - Return `"\n".join(lines)`.

3. If `model` is falsy:
   - Return `_build_structured_report(...)`.
   - `_build_structured_report`:
     - Builds title and summary.
     - If `retracted_premises` is truthy, adds a `## Fixed Defects (Retracted Premises)` table sorted by descending `dependent_count`.
     - If `blockers` is truthy, adds `## Active Blockers`, sorted by descending number of gated beliefs. For each blocker it writes the blocker ID, text, gated-belief count, total dependent count from `blocker_details`, and a sorted bullet list of gated beliefs by ID.
     - If there are no blockers and no retracted premises, adds `No blockers or retracted premises found.`
     - Always adds a `## Methodology` section.
     - Returns the joined Markdown.

Note: the structured report uses the heading `## Active Blockers`, while the LLM prompt asks for an `"Open Problems"` section. The code does not enforce the LLM’s output structure.

## Side Effects

- Reads the current local date via `date.today().isoformat()`, so output changes by day/timezone.
- In LLM mode, lazily imports `.llm` inside `generate_report` and calls `invoke_model`, which may perform network I/O, incur latency/cost, and produce non-deterministic output.
- Does not mutate the input dictionaries/lists.
- Does not write files, print to stdout, or maintain global state.
- No caching; every truthy `model` call performs another LLM invocation.

## Error Handling

There is no `try/except` and no fallback from LLM mode to structured mode.

Exceptions propagate to the caller, including:

- `KeyError` for missing required keys such as `gated_data["blocker_count"]`, `gated_data["gated_count"]`, `bdata["gated"]`, `g["id"]`, etc.
- `AttributeError` if `blocker_details[bid]` exists but is not a dict, or if `retract_reason` is not a string in structured mode.
- `TypeError` for incompatible data shapes, e.g. `dependent_count=None` during sorting, a generator passed as `retracted_premises` on the LLM path due to `len(...)`, or a non-string LLM response passed to `"\n".join(...)`.
- Any exception raised by `invoke_model`, such as timeout, authentication failure, provider error, or invalid model name.

The caller is responsible for validating inputs and handling LLM failures.

## Usage Patterns

Typical callers:

- CLI/command layer that has already collected gated beliefs, retracted premises, blocker details, and IN/OUT counts.
- Tests that call `report_gated(..., model="")` to assert deterministic Markdown.
- Tests that mock `generate_report` or `invoke_model` for the LLM path.

Caller obligations:

- Provide all required `gated_data` keys.
- Keep summary counts consistent with the detailed sections.
- Leave `model` empty for deterministic structured output.
- Pass a valid model name and safe timeout when using LLM mode.
- Handle exceptions from the LLM layer explicitly.

## Dependencies

- `datetime.date` from the standard library.
- Internal helpers in the same module:
  - `_format_data_for_prompt`
  - `_build_structured_report`
  - `generate_report`
  - `REPORT_GATED_PROMPT`
- Lazy dependency on `reasons.llm.invoke_model` through `generate_report`.

## Topics to Explore

- [function] `reasons/report_gated.py:_format_data_for_prompt` — Builds the exact text payload the LLM sees, including which fields are included or omitted.
- [function] `reasons/report_gated.py:_build_structured_report` — Implements the deterministic Markdown sections used when no model is supplied.
- [function] `reasons/report_gated.py:generate_report` — Wraps prompt construction and the `invoke_model` call where LLM failures originate.
- [file] `reasons/llm.py` — Defines `invoke_model`, the real external dependency for the LLM path.
- [file] `tests/test_report_gated.py` — Shows expected Markdown output and likely mocking strategy for both report modes.

## Beliefs

- `report-gated-mode-selected-by-model-truthiness` — `report_gated` takes the LLM path whenever `model` is truthy and the structured path whenever `model` is falsy.
- `report-gated-requires-gated-count-keys` — Both paths read `gated_data["blocker_count"]` and `gated_data["gated_count"]`, so omitting either key raises `KeyError`.
- `report-gated-does-not-fallback-on-llm-errors` — `report_gated` has no exception handling around `generate_report`, so LLM/timeout failures propagate to the caller.
- `report-gated-structured-report-sorts-by-impact` — The structured report sorts retracted premises by descending `dependent_count` and blockers by descending count of gated beliefs.
- `report-gated-summary-counts-are-not-recomputed` — The summary line uses `gated_data["blocker_count"]` and `gated_data["gated_count"]` directly rather than deriving them from the rendered sections.
