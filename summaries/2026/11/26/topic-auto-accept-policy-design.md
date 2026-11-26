---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Whether `source-divergence` should bypass the report entirely or just record a status flip; the choice affects whether you gate in `_store_proposals` (row state) vs. `cmd_propose_update` (report output).

## What the Code Actually Shows

### 1. `source-divergence` exists as a *tag*, not as *control flow*

The only structural facts about `source-divergence` in the observed code are:

- It is a member of the `BASES` set in `reasons/propose_update.py:25`.
- It is described in the LLM prompt (`reasons/propose_update.py:52`) as *"the source file/URL changed (mechanically verifiable, cheap to accept)"*.
- It is one of three choices in the CLI (`reasons/cli.py:3351, 3365, 3383, 3394`).

Crucially, **nothing in `propose_update.py` branches on `basis`**. In `parse_update_proposals` the basis is validated (defaults to `prior-knowledge` when invalid — see `test_invalid_basis_defaults` at `tests/test_propose_update.py:215`), and in `format_proposal_markdown` / `format_proposals_file` it is rendered as a label but never changes behavior. The only branch in `format_proposal_markdown` is `action == "UPDATE"` vs. `"RETRACT"`. Every test that uses `source-divergence` (`tests/test_propose_update.py:172, 184, 225, 233, 244, 278, 293, 335, 348, 369, 443`; `tests/test_api.py:1008, 1014, 1513, 1612, 1623`) treats it identically to any other basis — it round-trips through parse → format → propose unchanged.

So today: `basis` is a **descriptive tag**, not a **control-flow discriminator**. Whatever choice you make about "bypass vs. status flip," you are *adding* branching that does not exist yet.

### 2. The proposal path is deliberately manual

`format_proposal_markdown` (in `reasons/propose_update.py`) emits a suggested accept command for every proposal:

> ```
> **Command:** `reasons update {nid} "{escaped}"`
> **Command:** `reasons retract {nid} --reason "{fm}: {comment}"`
> ```

The prompt (line ~52) tells the model *"Let the human reviewer decide"* — this is the whole point of the proposal/report. A proposal is not a state mutation; it is a *recommendation* that a later `reasons update` or `reasons retract` command materializes.

That means placing `source-divergence` handling in **row state** (`_store_proposals`) would short-circuit the propose → accept separation the module is built around, *unless* the bypass path re-implements the accept step itself.

### 3. What's missing from the observations

Several things the question assumes simply are **not present** in the gathered data:

- **`_store_proposals` was not found** (`find_store_proposals_definition` → `count: 0`). The search only looked for `def _store_proposals`, so it may be a nested/private helper the search missed, or it may live in a module not in the observation set. You cannot cite it or reason about it from these observations.
- **`cmd_propose_update` body is not shown** — only its definition line at `reasons/cli.py:2775`. There is no evidence of how it currently handles the proposal list, so we cannot say whether basis-based filtering already exists there.
- **No `proposed-entries` / `report.json` artifact exists** (`total_count: 0`). So there is no persisted "proposals store" to gate against; the report is generated ad-hoc from the LIP node graph.
- **`reasons/report_gated.py` is a *different* report** — it consumes `gated_data` and `retracted_premises` (already-committed OUT nodes with `retract_reason` metadata). It does **not** reference `propose_update`, `basis`, or `source-divergence` at all. So it is not the "report output" surface being debated here — that surface is `cmd_propose_update` (or whatever renders `format_proposals_file`).

### 4. Weighing the two gates

**Gate in `_store_proposals` (row state):**

- The flip is persistent and visible to everything downstream — `reasons list-gated`, `reasons show`, cascade computation, and `report_gated.py` (which reads `retract_reason` metadata from OUT premises).
- But: `propose_update.py` performs **no verification** that the source actually changed. The LLM simply *labels* a proposal `source-divergence`. The "mechanically verifiable" claim in the prompt is aspirational — there is no fingerprint, hash, or URL-diff check in this module. Writing row state based on an unverified label is unsafe.
- Also: recording a row flip on a *proposal* conflates proposal with commit, and would need to either emit a real `retract_reason` (so `report_gated.py` picks it up as a "Fixed Defect") or introduce a new status the codebase doesn't currently model.

**Gate in `cmd_propose_update` (report output):**

- Stays within the existing contract: proposals remain proposals; the human still issues `reasons update`/`reasons retract`.
- Allows treating `source-divergence` as a *priority class* in the rendered report — e.g., "Auto-acceptable (source changed)" section, or excluded from the "needs scrutiny" list — without touching the belief store.
- Requires no new `_store_proposals` machinery, and no changes to `parse_update_proposals` validation.

### 5. Recommended reading

Given that:

1. Code today has **no verification** of `source-divergence` — the "cheap to accept" label is unbacked;
2. `format_proposal_markdown` consistently emits a manual accept command;
3. `_store_proposals` is not even observable in the codebase snapshot;
4. `report_gated.py` is a separate, downstream consumer of *committed* retractions, not proposals;

…**`source-divergence` should be surfaced/differentiated in `cmd_propose_update`'s report output, not used to bypass into row state in `_store_proposals`.**

The strongest version of this is: keep `source-divergence` as a proposal, but have `cmd_propose_update` group/annotate/badge it in the markdown so reviewers can accept it in one pass. If the team eventually wants true bypass, the flip belongs in the *accept* path (`reasons retract` / `reasons update`) after a real mechanical diff check is added — and only then would `_store_proposals` (or whatever writes the node's `metadata`/`truth_value`) be the correct home. Doing it earlier means an LLM label — a heuristic — is silently mutating the belief store, which contradicts the "proposal" contract the module documents.

## Topics to Explore

- [file] `reasons/cli.py` — Read `cmd_propose_update` at line 2775; this is the actual report-output gate and is not shown in the observations.
- [function] `reasons/cli.py:cmd_propose_update` — Confirm whether any existing basis-based filtering or grouping already happens before rendering `format_proposals_file`.
- [general] `_store_proposals-location` — The observation set contains no definition; locate it (likely a nested helper in `reasons/cli.py`) before deciding whether row-state gating is even possible.
- [file] `reasons/report_gated.py` — Verify how `retracted_premises` are gathered (does it read `retract_reason` metadata directly?) to understand downstream consequences of any status flip.
- [general] `source-divergence-verification` — There is no hash/fingerprint/URL-diff logic anywhere in `propose_update.py`; check whether any other module verifies that a source actually changed before treating the label as trustworthy.

## Beliefs

- `source-divergence-is-a-basis-value` — `source-divergence` is one of three members of the `BASES` set declared at `reasons/propose_update.py:25`.
- `basis-is-not-a-control-flow-discriminator` — Neither `parse_update_proposals`, `format_proposal_markdown`, nor `format_proposals_file` branches on the value of `basis`; it is validated and rendered but never changes behavior.
- `proposal-markdown-requires-manual-accept` — `format_proposal_markdown` emits a `reasons update <id> ...` or `reasons retract <id> ...` command for every proposal, so proposals never auto-commit in the observed code.
- `store-proposals-is-unobservable` — No `_store_proposals` definition was found in the observation set, so any row-state gating claim cannot be verified against current code.
- `report-gated-ignores-proposals` — `reasons/report_gated.py` consumes already-committed `retracted_premises` and `gated_data`, and contains no reference to `propose_update`, `basis`, or `source-divergence`.
