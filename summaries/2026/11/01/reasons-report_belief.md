---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/report_belief.py

# `reasons/report_belief.py`

## 1. Purpose

This module is the **reporting layer** for a single belief in a Truth Maintenance System (TMS). Given a belief node, its justification chain (the derivation steps from rules to premises), the root premises, and optional source evidence chunks, it produces a **human-readable markdown report** that traces *why* the belief is held (or not held).

It owns two distinct rendering strategies:

- **Deterministic** (default): A purely mechanical, structured markdown document built from the data. No external service calls.
- **LLM-synthesized** (when `model` is specified): The same data is formatted into a prompt, sent to an LLM, and the narrative prose is wrapped in a standard header.

The module is a **leaf renderer** — it does not mutate state, does not persist anything, and has no side effects beyond the optional LLM call.

## 2. Key Components

### `REPORT_BELIEF_PROMPT` (module-level constant)
A system prompt template with a single `{data}` placeholder. It instructs the LLM to:
- Write professional prose explaining the belief and its status
- Walk the justification chain step-by-step
- Present source evidence per premise
- Highlight OUT premises or missing evidence
- **Not** invent information, and **not** include a title (the caller adds one)

The `{data}` slot is filled by `_format_data_for_prompt`.

### `_build_structured_report(node_id, node_detail, explain_steps, premises_data, source_chunks)`
The deterministic renderer. Returns a markdown string. Contract:

| Param | Type | Role |
|---|---|---|
| `node_id` | `str` | The belief's identifier, used in the `# Belief Report:` heading |
| `node_detail` | `dict` | Keys: `truth_value`, `text`, `source` (all optional via `.get`) |
| `explain_steps` | `list[dict]` | Ordered derivation steps; keys: `node`, `truth_value`, `reason`, `antecedents`, `label`, `outlist` |
| `premises_data` | `list[dict]` | Root premises; keys: `id`, `truth_value`, `text`, `source` |
| `source_chunks` | `dict[str, list[dict]] | None` | Maps premise-id → list of evidence chunks (`filename`, `section`, `text`) |

Output structure:
```
# Belief Report: `<node_id>`
*Generated <date> — status: <truth_value>*
## Belief
## Justification Chain
## Root Premises (N)
  ### [+/-] `<premise_id>`
  **Source Evidence:** (collapsible <details> blocks)
```

The `[+]` / `[-]` marker is a simple TMS truth-value indicator: `+` for IN, `-` for everything else (OUT, UNKNOWN).

### `_format_data_for_prompt(node_id, node_detail, explain_steps, premises_data, source_chunks)`
Flattens the same data into a **plain-text / light-markdown** string suitable for stuffing into the LLM prompt. Notably, it does **not** include the report title or the "Generated" date line — those are added by `report_belief` after the LLM returns.

### `generate_report(data_text, model, timeout)`
Thin wrapper around `invoke_model`. Performs the **lazy import** of `.llm` inside the function body, so the LLM dependency is only loaded when actually needed.

### `report_belief(node_id, node_detail, explain_steps, premises_data, source_chunks, model="", timeout=300)`
The **public entry point**. Decision logic:

```
if model is truthy:
    data_text = _format_data_for_prompt(...)
    content   = generate_report(data_text, model, timeout)   # LLM call
    return "<header>\n" + content
else:
    return _build_structured_report(...)                       # deterministic
```

Returns a `str` (markdown) in **both** branches. The LLM path prepends the `# Belief Report` heading and the `*Generated … status …*` line, because the prompt explicitly told the model not to include a title.

## 3. Patterns & Conventions

- **Dual-mode rendering with a single entry point.** The `model=""` default keeps the common path (no LLM) synchronous and side-effect-free. The LLM path is opt-in.
- **Lazy import for optional heavy dependencies.** `from .llm import invoke_model` lives inside `generate_report`, not at module top level. This means `report_belief.py` can be imported (and the deterministic path used) without the LLM stack being installed or configured.
- **Defensive `.get()` with defaults.** Every dict access uses `.get(key, default)` rather than `[key]`, so partially-populated data (e.g., a premise with no `source`) renders gracefully instead of raising `KeyError`.
- **HTML `<details>` for collapsible evidence.** The structured report embeds raw HTML (`<details><summary>…</summary>…</details>`) inside the markdown. This works in GitHub-flavored markdown and most IDE previewers.
- **Mirror structure between the two formatters.** `_build_structured_report` and `_format_data_for_prompt` traverse the same data in the same order (belief → chain → premises → chunks). If you add a new field to one, you should add it to the other.
- **`reasons/` vs `reasons_lib/` duplication.** The repo has parallel `reasons/` and `reasons_lib/` packages. `reasons_lib/report_belief.py` is imported by `reasons/api.py` (per the "Imported By" list), suggesting `reasons/` is the public/CLI-facing layer and `reasons_lib/` is the library core. This file in `reasons/` appears to be the canonical implementation that `reasons_lib` re-exports or delegates to.

## 4. Dependencies

**Imports (what this file needs):**
- `datetime.date` — for the `*Generated YYYY-MM-DD*` line
- `reasons.llm.invoke_model` — LLM call (lazy, only in LLM path)

**Imported by (what needs this file):**
- `reasons/api.py` — the API layer calls `report_belief` to generate reports for HTTP/MCP consumers
- `reasons_lib/report_belief.py` — the library-parallel copy (likely a re-export or thin wrapper)
- `tests/test_report_belief.py` — unit tests for both rendering paths

## 5. Flow (Control & Data)

```
Caller (e.g., api.py)
  │
  │  has: node_id, node_detail, explain_steps, premises_data, source_chunks
  │  optionally: model name
  ▼
report_belief(...)
  │
  ├─ model is falsy ──────────────────────────────────────────────┐
  │                                                               ▼
  │                                              _build_structured_report(...)
  │                                                               │
  │                                              Builds markdown list-of-strings:
  │                                              • Header + date + status
  │                                              • Belief text + source
  │                                              • Justification chain (each step
  │                                                with [+/-], reason, antecedents,
  │                                                label, outlist)
  │                                              • Root premises (each with text,
  │                                                source, collapsible evidence)
  │                                                               │
  │                                                               ▼
  │                                                        "\n".join(lines)  →  return str
  │
  └─ model is truthy ─────────────────────────────────────────────┐
                                                                  ▼
                                               _format_data_for_prompt(...)
                                                                  │
                                               Flattens to plain text
                                               (same data, no HTML,
                                                no title/date)
                                                                  │
                                                                  ▼
                                               generate_report(data_text, model, timeout)
                                                                  │
                                               from .llm import invoke_model   ← lazy
                                               invoke_model(prompt, model, timeout)
                                                                  │
                                                                  ▼
                                                           LLM response (str)
                                                                  │
                                                                  ▼
                                               Prepend "# Belief Report: …"
                                               Prepend "*Generated … status …*"
                                               "\n".join → return str
```

Data transformation summary: **structured dicts → markdown string**. The deterministic path preserves structure; the LLM path converts to prose. Both end up as a single markdown `str`.

## 6. Invariants

- **Return type is always `str`.** Neither branch returns `None`, a list, or raises on "empty" data. An empty `explain_steps` list yields `"No justification chain available."`; an empty `premises_data` yields `"No root premises found."`
- **`node_id` appears in every output** as the report title (`# Belief Report: \`{node_id}\``).
- **Truth-value markers are binary.** `+` iff `truth_value == "IN"`, otherwise `-`. There is no distinct marker for OUT vs. UNKNOWN — both render as `-`.
- **`source_chunks` may be `None`.** The structured report checks `source_chunks is not None and pid in source_chunks` to distinguish "chunks were not requested" from "chunks were requested but empty." In the prompt formatter, `source_chunks.get(pid, [])` is used, which would raise `AttributeError` if `source_chunks` were `None` — a potential inconsistency.
- **The LLM must not produce a title.** The prompt explicitly forbids it, and `report_belief` prepends the canonical title. If the model ignores the instruction, you get a duplicated title.
- **Date is wall-clock at call time.** `date.today()` is called inside the rendering functions, so the "Generated" stamp reflects report-generation time, not belief-creation time.

## 7. Error Handling

This module is **deliberately thin on error handling** because it is a pure renderer:

- **No try/except anywhere.** If `invoke_model` raises (timeout, API error, bad model name), the exception propagates directly to the caller. The `timeout=300` parameter is the only safeguard.
- **Missing keys are tolerated** via `.get(key, default)` — a `node_detail` without `truth_value` renders as `UNKNOWN` rather than crashing.
- **`source_chunks = None`** is handled in `_build_structured_report` (guarded by `is not None`) but **not** in `_format_data_for_prompt`, where `.get()` would raise `AttributeError` on `None`. In practice the caller (`api.py`) presumably always passes a dict or an empty dict, but this is a latent bug if a `None` slips through.
- **No input validation.** There is no check that `node_id` is non-empty, that `premises_data` items actually contain an `id` key, etc. The module trusts its caller.

The design philosophy: *let the caller validate; this module just formats.*

---

## Topics to Explore

- [file] `reasons/api.py` — Shows how `report_belief` is called in practice: what shape `explain_steps` and `premises_data` actually take, and how the `model` parameter is wired from the request layer.
- [function] `reasons/llm.py:invoke_model` — The actual LLM invocation: how `model` is resolved, what timeout semantics apply, and what exception types can propagate back into `report_belief`.
- [file] `tests/test_report_belief.py` — The test suite reveals the expected output shape for both the structured and LLM paths, including edge cases like empty premises or missing source chunks.
- [file] `reasons_lib/report_belief.py` — The parallel `reasons_lib` copy; understanding how it relates to `reasons/report_belief.py` (re-export? fork? thin wrapper?) clarifies the dual-package architecture.
- [general] `tms-truth-values` — The broader TMS semantics of IN/OUT truth values, the `outlist` concept in justification steps, and how they map to the `[+]`/`[-]` markers in the report.

---

## Beliefs

- `report-belief-dual-mode` — `report_belief` produces a deterministic structured markdown report when `model` is falsy, and an LLM-synthesized narrative (wrapped in the same header) when `model` is truthy; the return type is `str` in both cases.
- `lazy-llm-import` — `invoke_model` is imported inside `generate_report` (not at module scope), so importing `reasons.report_belief` and using the deterministic path does not require the LLM stack to be installed.
- `source-chunks-none-inconsistency` — `_build_structured_report` guards against `source_chunks is None`, but `_format_data_for_prompt` calls `source_chunks.get(pid, [])` without a `None` check, making the LLM path vulnerable to a `TypeError` if the caller passes `None` instead of `{}`.
- `report-belief-owns-no-state` — The module is a pure function of its inputs: it reads no storage, writes no files, and has no side effects beyond the optional LLM network call; all persistence and data gathering is the caller's responsibility.
- `truth-value-marker-is-binary` — The `[+]` / `[-]` prefix in both report renderers distinguishes only between `truth_value == "IN"` and everything else (OUT, UNKNOWN, missing); there is no three-way marker.
