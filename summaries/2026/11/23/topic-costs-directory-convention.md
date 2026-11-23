---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: how `costs/*.json` files are named, rotated, consumed, and whether any script validates the aggregate-vs-breakdown invariant.

# How `costs/*.json` files are named, rotated, consumed, and validated

## Naming

The only concrete evidence is the sample file `costs/2026-11-19T180331-update.json` (inspected via `inspect_existing_cost_file_schema`). Its filename decodes as:

```
costs/<timestamp-with-colons-stripped>-<operation>.json
```

- `<timestamp>` = `2026-11-19T180331` — ISO 8601 with the `HH:MM:SS` colons removed, presumably for filesystem portability (Windows forbids `:` in filenames). Inside the file, the `"timestamp"` field retains colons: `"2026-11-19T18:03:31"`.
- `<operation>` = `update` — matches the JSON body's `"operation": "update"` field.

So each file name is derived from the same two values it stores: `timestamp` and `operation`. This strongly implies that the writer (not directly observed) formats the name as `f"{ts.replace(':', '')}-{operation}.json"`. Whether other operations (`add`, `retract`, etc.) produce sibling files such as `…-add.json` is not shown — only the `-update` variant was sampled.

## Rotation

There is **no evidence of rotation logic**. The pattern search `find_cost_rotation_or_update_naming` matched 47 lines, but every match is about unrelated `timestamp` handling in `reasons/api.py` (e.g. line 2186 "Restore exact timestamps AFTER retract/assert", line 4173 `"timestamp": now`), `reasons/cli.py` (history/report timestamps), `reasons/pg.py` (propagation-log timestamps), etc. None reference cost files, `update.json`, or rotation semantics.

What the naming scheme *implies* (but is not directly confirmed by code):

- **Per-operation append-only files.** Because each file embeds a timestamp, every operation likely writes a *new* file, so accumulation across time happens by filename ordering, not by rewriting a single file. That is closer to "versioned append" than classic log rotation (rename/compress).
- **Unclear whether old files are ever pruned, compressed, or capped.** No cleanup code appears in the observations.

## Consumption

The only consumer hint is in `reasons/llm.py`:

- Line 11: `"counts and costs. Use get_cost_summary() to retrieve accumulated stats."`
- Line 124: `"Parse JSON output from CLI, extract response text and record costs."`

So `reasons/llm.py` is where costs are *recorded* (writer side), and `get_cost_summary()` is the *reader* accessor for accumulated stats. Neither the writer that constructs the filename nor the implementation of `get_cost_summary()` is captured in these observations, so the exact glob / iterating logic ("read every `costs/*.json` and sum") cannot be confirmed.

Notable: `.gitignore` (21 lines, checked via `check_whether_costs_are_ignored`) does **not** ignore `costs/` or `*.json`, so these files are committed to the repo. That makes them a durable, reviewable ledger rather than transient artifacts — which is why an aggregate-vs-breakdown invariant matters.

## Aggregate-vs-breakdown invariant

**Structure** (from the sample file):

- **Aggregate fields** — top-level: `calls`, `input_tokens`, `output_tokens`, `cost_dollars`.
- **Breakdown field** — `by_model`, a map keyed by model identifier, each entry holding `calls`, `input_tokens`, `output_tokens`, `total_cost_usd`.
- Also a redundant single-model convenience field `model`, which duplicates the sole key in `by_model` when only one model is used.

In the sampled file the invariant holds trivially: `by_model` has exactly one entry (`openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017`) whose numeric values exactly match the top-level aggregates.

**Validation** — none found:

- `find_aggregate_breakdown_logic` — `aggregate|breakdown` → **0 matches**.
- `find_cost_docs_and_workflow_mentions` — `costs/|aggregate|breakdown` → **0 matches**.
- `find_cost_validation_in_tests` — `cost|aggregate|breakdown` → **0 matches**.
- `find_ci_cost_checks` — `cost|aggregate|breakdown` → **0 matches**.

Combined with the fact that no file-content loader for `costs/*.json` was found by `find_cost_file_handling` (`costs|cost_file|cost_path|costs_dir` produced only two `reasons/llm.py` docstring hits), the conclusion is:

> **No script in the observed scope validates that `sum(by_model[*].{calls,input_tokens,output_tokens,total_cost_usd}) == {calls,input_tokens,output_tokens,cost_dollars}` for each file.**

Caveat: a zero-match is weaker than a proof. The invariant could be checked without using the words "aggregate" or "breakdown" (e.g., `assert sum(m["calls"] for m in by_model.values()) == calls` or via `pydantic` model validators that reference `by_model`). The search patterns were narrow, and the actual writer/reader implementation in `reasons/llm.py` was not captured beyond two docstring lines, so this claim should be re-verified directly against that file.

## What's missing from the observations

- The **writer** in `reasons/llm.py` that formats the filename and populates `operation`.
- The **implementation** of `get_cost_summary()` (does it glob `costs/*.json`? does it cache? does it key by domain?).
- Any **pruning/cleanup** path for old cost files.
- A full listing of `costs/` to see which `-<operation>` suffixes exist.
- Whether `model` (top-level) is expected to equal the `by_model` key when only one model is used, or only the "dominant" model otherwise.

## Topics to Explore

- [file] `reasons/llm.py` — The cost-recording module; locate the code that builds `f"{ts}-{operation}.json"` and implements `get_cost_summary()`.
- [function] `reasons/llm.py:get_cost_summary` — Determines how `costs/*.json` files are discovered, parsed, and combined into accumulated stats.
- [general] `cost-file-glob` — Search for `glob("costs/*.json")`, `os.listdir("costs")`, or `Path("costs")` to confirm the consumer's discovery mechanism.
- [general] `aggregate-vs-breakdown-check` — Grep tests and CI for `sum(... by_model ...)` or pydantic validators on the cost schema to definitively rule in/out invariant enforcement.
- [file] `costs/` — List the directory to enumerate the operation suffixes actually emitted (e.g. `-update`, `-add`, `-retract`) and any auxiliary files.

## Beliefs

- `cost-filenames-produce-timestamped-per-operation-json` — Each cost file is named `costs/<ISO-timestamp-without-colons>-<operation>.json`, as evidenced by `2026-11-19T180331-update.json` whose body has `timestamp: "2026-11-19T18:03:31"` and `operation: "update"`.
- `costs-directory-is-git-tracked` — `.gitignore` does not list `costs/` or a `*.json` pattern, so cost JSON files are committed rather than treated as transient output.
- `by-model-maps-to-top-level-aggregates` — In each cost file, `by_model[*].{calls,input_tokens,output_tokens,total_cost_usd}` is the per-model breakdown corresponding to the top-level `calls`/`input_tokens`/`output_tokens`/`cost_dollars` aggregates.
- `get-cost-summary-returns-accumulated-costs` — `reasons/llm.py:11` documents `get_cost_summary()` as the accessor for accumulated cost statistics.
- `no-script-validates-aggregate-vs-breakdown` — Searches for `aggregate|breakdown` in code, tests, CI, and docs returned zero matches, as did `cost|aggregate|breakdown` in tests and CI.
