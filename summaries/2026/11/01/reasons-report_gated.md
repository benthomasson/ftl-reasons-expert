---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/report_gated.py

# `reasons/report_gated.py` — Gated Beliefs Report Generator

## 1. Purpose

This module is the **reporting layer** for the "gated beliefs" domain in a Truth Maintenance System (TMS). It owns one job: turn a pre-gathered snapshot of the belief network's problem state into a human-readable markdown document.

Two categories of "problems" are in scope:

- **Retracted premises** — beliefs that were once IN but have been retracted (i.e., defects that were *fixed*). The report treats these as "Fixed Defects."
- **Active IN blockers** — beliefs that remain IN but gate (block) other beliefs from being justified. These are the "Open Problems."

The module does **not** gather data from storage. By the time `report_gated()` is called, all data (`gated_data`, `retracted_premises`, `blocker_details`, `in_count`, `out_count`) has already been collected upstream (likely in `reasons/api.py`). This file is purely a **presentation/serialization** concern.

## 2. Key Components

### `REPORT_GATED_PROMPT` (module-level constant)

A string template with a single `{data}` placeholder. It instructs an LLM to produce a professional markdown report with specific sections (Summary, Fixed Defects, Open Problems, Impact Analysis). Notably it says:

> "Do NOT include a title — the report already has one"

This is a critical contract: the caller (`report_gated`) prepends the `# Gated Beliefs Report` header and a metadata line, so the LLM must not duplicate them.

### `_build_structured_report(in_count, out_count, gated_data, retracted_premises, blocker_details)`

The **deterministic** report generator. No LLM involved. Returns a markdown string.

Contract:
- Always produces a `# Gated Beliefs Report` H1.
- Emits a metadata line with date, IN/OUT counts, blocker/gated counts.
- If `retracted_premises` is non-empty → renders a markdown table sorted by `dependent_count` descending.
- If `blockers` dict is non-empty → renders numbered `###` subsections sorted by number of gated beliefs descending.
- If **both** are empty → emits "No blockers or retracted premises found."
- Always appends a `## Methodology` section referencing the CLI commands used.

### `_format_data_for_prompt(in_count, out_count, gated_data, retracted_premises, blocker_details)`

Serializes the same data into a **plain-text structured format** suitable for embedding in the LLM prompt. No markdown formatting (no tables, no `**bold**`). Uses `###` sub-headers and indented lists. This is the "data blob" that fills `{data}` in `REPORT_GATED_PROMPT`.

### `generate_report(data_text, model, timeout)`

Thin wrapper around `invoke_model`. Constructs the final prompt by formatting the template, then calls the LLM. Returns the raw LLM string output.

### `report_gated(gated_data, retracted_premises, blocker_details, in_count, out_count, model="", timeout=300)`

The **public entry point** (imported by `reasons/api.py`). Dispatches:

| `model` parameter | Path taken |
|---|---|
| `""` (default) | `_build_structured_report` — pure string assembly |
| non-empty string | `_format_data_for_prompt` → `generate_report` → prepend header |

The LLM path **always** prepends a `# Gated Beliefs Report` title and the metadata line, then appends the LLM's narrative content.

## 3. Patterns

- **Strategy / Two-mode dispatch**: The `model` parameter acts as a strategy selector. The same logical report can be produced deterministically or via LLM, with the caller choosing at the boundary.
- **Lazy import**: `from .llm import invoke_model` is inside `generate_report()`, not at module top-level. This avoids importing the LLM stack (and its dependencies like `httpx`, API keys, etc.) when only the structured path is needed. It also prevents circular-import issues if `.llm` transitively imports back into the `reasons` package.
- **Presentation separation**: Data gathering (upstream in `api.py`) is cleanly separated from data formatting and report assembly (this file). The data structures passed in are plain dicts/lists — no ORM objects, no async.
- **Sort-before-render**: Every collection is sorted deterministically (retracted premises by `dependent_count` desc; blockers by gated-count desc; gated beliefs by ID). This ensures stable, reproducible output in the structured path.
- **Template + slot**: `REPORT_GATED_PROMPT` uses Python's `str.format` with a single `{data}` slot rather than f-strings, making it a reusable template.

## 4. Dependencies

### Imports (this file depends on)

| Import | Where | Why |
|---|---|---|
| `datetime.date` | top-level | Formatting the generation date in the metadata line |
| `.llm.invoke_model` | **lazy**, inside `generate_report` | Calling the LLM to synthesize narrative |

### Imported by (depends on this file)

| Consumer | Likely usage |
|---|---|
| `reasons/api.py` | Calls `report_gated()` after gathering gated data from storage |
| `reasons_lib/report_gated.py` | Mirrored copy for the library distribution (the repo has a `reasons/` source and a `reasons_lib/` packaged variant) |
| `tests/test_report_gated.py` | Unit/integration tests exercising both the structured and LLM paths |

## 5. Flow (Control Path)

```
Caller (api.py)
  │
  │  gathers: gated_data, retracted_premises, blocker_details, in_count, out_count
  │
  ▼
report_gated(...)
  │
  ├── model == ""  ──────────────────────────────────────────────┐
  │                                                              │
  │    _build_structured_report(...)                              │
  │      1. Emit H1 + metadata line                               │
  │      2. If retracted_premises → sort, render markdown table   │
  │      3. If blockers → sort, render ### subsections            │
  │      4. If both empty → "No blockers…" message               │
  │      5. Emit Methodology section                               │
  │      6. Join lines, return string                              │
  │                                                              │
  ├── model != ""  ──────────────────────────────────────────────┐
  │                                                              │
  │    _format_data_for_prompt(...)                               │
  │      → plain-text data blob                                   │
  │                                                              │
  │    generate_report(data_text, model, timeout)                 │
  │      → REPORT_GATED_PROMPT.format(data=data_text)            │
  │      → invoke_model(prompt, model, timeout)                   │
  │      → returns LLM narrative string                            │
  │                                                              │
  │    Prepend H1 + metadata line to LLM output                   │
  │    Return combined string                                     │
  │                                                              │
  ▼
Markdown string returned to caller
```

## 6. Invariants

1. **Title contract**: The final output always begins with exactly one `# Gated Beliefs Report` line. In the LLM path, the prompt explicitly forbids the model from adding a title. If the LLM disobeys, the caller gets a duplicate H1 — this is a *soft* invariant enforced by prompt, not code.

2. **Deterministic ordering (structured path)**: Retracted premises are sorted by `dependent_count` descending; blockers by `len(gated)` descending; individual gated beliefs by `id` ascending. This makes the output stable and diff-friendly.

3. **Pipe-escaping in tables**: In `_build_structured_report`, the `text` and `retract_reason` fields are passed through `.replace("|", "\\|")` before being placed in a markdown table cell. Without this, any premise text containing a literal `|` would break the table layout.

4. **Truncation**: In the structured table, premise text is truncated to 80 characters (`rp["text"][:80]`). In the LLM-prompt data, text is **not** truncated — the LLM sees the full text.

5. **Empty-state guard**: If neither `retracted_premises` nor `blockers` exist, the structured path emits a single "No blockers or retracted premises found." line rather than two empty sections.

## 7. Error Handling

This file is notably **silent** on error handling:

- **No input validation.** If `gated_data` is missing the `"blocker_count"`, `"gated_count"`, or `"blockers"` keys, you get a `KeyError`. If `retracted_premises` items lack `"id"`, `"text"`, `"retract_reason"`, or `"dependent_count"`, you get a `KeyError`. The caller in `api.py` is responsible for constructing these dicts correctly.
- **LLM failures propagate.** If `invoke_model` raises (network timeout, API error, model not found), the exception bubbles up through `generate_report` → `report_gated` → caller. There is no try/except, no fallback to the structured path. The `timeout` parameter (default 300 s) is the only safeguard.
- **No logging.** There is no `logger` usage anywhere in this file. Debugging a malformed report requires reading the output.
- **The LLM may hallucinate.** The prompt says "Do NOT invent information not present in the data," but there is no post-hoc validation that the LLM output references only real belief IDs. A developer adding verification would need to cross-check IDs in the LLM output against the input data.

---

## Topics to Explore

- [file] `reasons/api.py` — The caller that gathers `gated_data`, `retracted_premises`, and `blocker_details` before invoking `report_gated`; shows how the data contracts are constructed and what storage queries back them.
- [function] `reasons/llm.py:invoke_model` — The LLM invocation primitive; understanding its timeout behavior, retry logic, and error semantics is critical since `report_gated` has no local error handling.
- [file] `tests/test_report_gated.py` — The test suite reveals the exact expected data shapes for `gated_data`, `retracted_premises`, and `blocker_details`, and shows how the LLM path is mocked.
- [file] `reasons/report_belief.py` — A sibling report generator for individual beliefs; comparing the two shows the project's reporting conventions and whether there's shared formatting logic that could be extracted.
- [general] `reasons-vs-reasons_lib-mirroring` — The repo maintains parallel `reasons/` and `reasons_lib/` trees; understanding the build/packaging flow (likely in `pyproject.toml`) clarifies whether changes to one file must be mirrored to the other.

## Beliefs

- `report-gated-uses-lazy-import-for-llm` — `invoke_model` is imported inside `generate_report()` rather than at module level, so the structured-report path has zero LLM-stack dependencies at import time.
- `report-gated-llm-path-prepends-title` — When a model is specified, `report_gated` always prepends `# Gated Beliefs Report` and a metadata line to the LLM output; the LLM prompt explicitly forbids the model from generating its own title.
- `report-gated-structured-path-sorts-by-impact` — In the deterministic path, retracted premises are ordered by `dependent_count` descending and blockers by gated-belief count descending, making the most impactful items appear first.
- `report-gated-has-no-input-validation` — `report_gated` does not validate the structure of `gated_data`, `retracted_premises`, or `blocker_details`; missing keys produce unhandled `KeyError` exceptions rather than graceful errors.
- `report-gated-pipes-escaped-in-tables` — The structured report escapes literal `|` characters in premise text and retract reasons (`replace("|", "\\|")`) to prevent markdown table breakage, but does not escape other markdown metacharacters.
