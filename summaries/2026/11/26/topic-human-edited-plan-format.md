---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: The observed code writes markdown with suggested commands but does not show how an edited plan file is parsed and applied.

## What the observation means

The observation is accurate for the **proposed-update workflow** shown in `reasons/propose_update.py`:

- `format_proposal_markdown()` renders each proposal as markdown, including current text, proposed text, failure mode, basis, evidence, cascade impact, and a **suggested shell command**.
- `format_proposals_file()` writes a review document headed `# Proposed Updates` and says:

  > “Review each proposal below. Accept by running the suggested command.”
  > — `reasons/propose_update.py:273`

- For updates, the suggested command is shaped like `reasons update <id> "<new text>"`; for retractions, it is shaped like `reasons retract <id> --reason "<failure-mode>: <comment>"`.

So this file is a **human review artifact with copy-paste commands**, not an editable plan format that the observed code re-reads and applies.

### The parse/apply machinery exists elsewhere

The observed code does contain explicit plan parsing and application for the **deduplication** and **contradiction** workflows:

| Workflow | Parser | Applier | CLI usage |
|---|---|---|---|
| Dedup | `api.parse_dedup_plan()` at `reasons/api.py:5107` | `api.apply_dedup_plan()` at `reasons/api.py:5139` | `reasons/cli.py:1437` reads the accept path, parses, then applies at `reasons/cli.py:1441` |
| Contradiction | `api.parse_contradiction_plan()` at `reasons/api.py:4867` | `api.apply_contradiction_plan()` at `reasons/api.py:4897` | `reasons/cli.py:2976` reads the accept path, parses, then applies at `reasons/cli.py:2980` |

Those are also exposed as CLI commands described as “Apply a reviewed dedup plan file” and “Apply a reviewed contradiction plan file” at `reasons/cli.py:3630` and `reasons/cli.py:3883`.

Tests confirm the intended round-trip for those two plan types:

- `tests/test_contradictions.py:506` tests `parse_contradiction_plan` plus apply.
- `tests/test_contradictions.py:530` tests `apply_contradiction_plan`.
- `tests/test_derive.py:801` tests writing and parsing a dedup plan.
- `tests/test_derive.py:824` tests the full deduplicate → write plan → parse → apply workflow.
- `tests/test_derive.py:851` tests `apply_dedup_plan`.

### Important distinction

`reasons/propose_update.py` does contain a function named `parse_update_proposals()`, but it parses **JSON returned by the LLM**, not an edited markdown plan file. It looks for a JSON array in the model response and normalizes fields like `action`, `failure_mode`, and `basis`. It does not parse the `**Command:**` lines emitted by `format_proposal_markdown()`.

So the correct reading is:

1. **Dedup and contradiction plan files** are written, parsed, and applied by explicit `parse_*_plan()` / `apply_*_plan()` functions.
2. **Proposed-update markdown** is written with suggested commands and is intended for manual command execution.
3. The observed code does **not** show an edited proposed-update markdown parser/applier.

### Limitation of the observations

The `propose_update.py` inspection returned the first 300 lines of a 332-line file. It is possible, though not shown, that the remaining 32 lines contain something relevant. However, the symbol search did not find any `parse_proposal_plan` or `apply_proposal_plan`-style matches, and the CLI matches for plan parsing/applying are tied to dedup and contradiction commands, not proposed updates.

## Topics to Explore

- [function] `reasons/api.py:parse_dedup_plan` — Shows the exact markdown grammar for edited dedup plan files and how `KEEP` / `RETRACT` entries are recognized.
- [function] `reasons/api.py:apply_dedup_plan` — Shows how parsed dedup clusters mutate the network and how missing nodes are handled.
- [function] `reasons/api.py:parse_contradiction_plan` — Shows how edited contradiction plans are parsed into actionable entries, including `APPLY` entries.
- [file] `reasons/cli.py` — Inspect the commands around lines 1437–1441 and 2976–2980 to see the full accept-plan CLI wiring.
- [file] `reasons/propose_update.py` — Read the full file, including lines after 300, to confirm whether any proposed-update plan parser/applier exists.

## Beliefs

- `dedup-plan-parse-and-apply-is-exposed-via-api` — `api.parse_dedup_plan` and `api.apply_dedup_plan` provide explicit parsing and application of reviewed dedup plan text.
- `contradiction-plan-parse-and-apply-is-exposed-via-api` — `api.parse_contradiction_plan` and `api.apply_contradiction_plan` provide explicit parsing and application of reviewed contradiction plan text.
- `propose-update-markdown-emits-commands-not-parsed-plan` — `format_proposals_file` writes markdown instructing reviewers to run suggested commands, and the observed code shows no parser/applier for that markdown.
- `parse-update-proposals-parses-llm-json-not-markdown` — `parse_update_proposals` extracts JSON proposals from LLM responses, not edited plan files.
- `cli-apply-reviewed-plan-commands-call-parse-then-apply` — The dedup and contradiction CLI flows read an accept path, call `parse_*_plan`, then call `apply_*_plan`.
