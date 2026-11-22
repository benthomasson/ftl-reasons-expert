---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: The prompt does not annotate `detected-contradiction` as cheap or scrutinized; check any design docs or downstream nogood handling to see whether it was intended for auto-accept or review.

The prompt is genuinely silent on how `detected-contradiction` should be handled. The observed code points to a **review-first** workflow for nogoods and update proposals, with auto-application only as an explicit opt-in. There is no observed evidence that `basis="detected-contradiction"` triggers auto-accept.

### 1. Prompt-level taxonomy does not classify `detected-contradiction`

In `reasons/propose_update.py`:

- `BASES` includes `"detected-contradiction"` at line 26.
- The basis description is just:
  - `"detected-contradiction: conflicts with another belief (nogood)"` at line 53.
- By contrast, the same prompt explicitly annotates:
  - `source-divergence` as “mechanically verifiable, cheap to accept”
  - `prior-knowledge` as “needs scrutiny — reviewer may share your blind spot”

So `detected-contradiction` is sandwiched between an explicitly cheap basis and an explicitly scrutinized basis, but itself has no cheap/scrutiny annotation. The prompt alone does not state whether it should be auto-accepted or reviewed.

### 2. Proposal rendering does not special-case `detected-contradiction`

`reasons/propose_update.py:parse_update_proposals` validates the basis value but does not map any basis to an auto-accept policy. If an invalid basis is returned, it defaults to `prior-knowledge`.

`reasons/propose_update.py:format_proposal_markdown` prints the basis and suggested command, but does not branch on basis.

`reasons/propose_update.py:format_proposals_file` renders all proposals with:

> “Review each proposal below. Accept by running the suggested command.”

That is a review-oriented output path for all bases, including `detected-contradiction`.

### 3. Nogood handling supports review by default and auto-apply as an explicit flag

The contradiction detector in `reasons/contradictions.py` uses an LLM prompt (`CONTRADICTION_PROMPT`) and `invoke_model` to propose nogoods. `detect_contradictions` returns parsed contradiction candidates and, when `output_path` is set, flushes them through `_flush_results` / `write_contradiction_plan`. It does not itself call `add_nogood`.

The tests show both modes:

- `tests/test_contradictions.py:test_output_writes_plan` expects CLI output to include `--accept` and the plan file to contain `### NOGOOD x-conflict [APPLY]`. That indicates a manual accept/review path.
- `tests/test_contradictions.py:test_auto_apply_calls_add_nogood` calls `api.detect_contradictions(auto_apply=True)` and asserts `applied >= 1`. That indicates auto-apply exists as an explicit opt-in.

`reasons/api.py:add_nogood` exists at line 1759 and returns details such as `nogood_id`, `changed`, and `backtracked_to`, but the observed tests show it is invoked through an explicit `auto_apply=True` path, not an implicit default from a basis label.

### 4. Design-doc evidence is missing

The docs search for:

`detected[-_]contradiction|auto[-_]accept|scrutiniz|cheap`

returned zero matches. The only design/skill doc found was `.claude/skills/reasons/SKILL.md`, but its contents were not read in the observations.

Also, the search for `detected-contradiction` found only:
- CLI choice lists in `reasons/cli.py` lines 3351, 3365, 3383, 3394
- the taxonomy definitions in `reasons/propose_update.py`
- one test reference in `tests/test_propose_update.py`

No observed code maps `basis="detected-contradiction"` to an auto-accept path.

### Conclusion

Based on the observations, the safest interpretation is:

- The **prompt itself does not classify** `detected-contradiction` as cheap or scrutinized.
- The **observed proposal workflow is review-first** for all bases.
- The **observed nogood workflow is review-first by default**, with `auto_apply=True` as an explicit opt-in.
- There is **no observed evidence** that `detected-contradiction` is intended for auto-accept.
- The exact intended policy is still unproven because the design doc `.claude/skills/reasons/SKILL.md` was not inspected, and the full implementation of `api.detect_contradictions` and CLI handling around lines 3351–3394 was not shown.

## Topics to Explore

- [file] `.claude/skills/reasons/SKILL.md` — The only design/skill doc found; likely place for intended review-vs-auto-accept policy.
- [function] `reasons/api.py:detect_contradictions` — Confirm default `auto_apply` behavior and whether severity or basis affects application.
- [general] `nogood-acceptance-policy` — Trace `write_contradiction_plan` accept handling versus `add_nogood` to verify whether plans require human acceptance.
- [function] `reasons/propose_update.py:format_proposals_file` — Verify all update proposals are rendered for review and no basis-specific bypass exists.
- [file] `reasons/cli.py` — Inspect lines around 3351–3394 to see how `detected-contradiction` is used as a CLI basis choice and whether any command auto-applies it.

## Beliefs

- `propose-update-basis-taxonomy-omits-detected-contradiction-cost` — In `reasons/propose_update.py`, `detected-contradiction` is in `BASES` and described only as “conflicts with another belief (nogood)” with no cheap/scrutiny annotation.
- `propose-update-renders-all-bases-for-review` — `format_proposals_file` instructs reviewers to accept proposals by running suggested commands and does not branch on `basis`.
- `contradiction-detection-writes-plan-not-nogood` — `detect_contradictions` returns parsed nogood candidates and optionally writes a plan via `write_contradiction_plan`; it does not call `add_nogood` directly.
- `api-detect-contradictions-has-explicit-auto-apply` — `tests/test_contradictions.py:test_auto_apply_calls_add_nogood` calls `api.detect_contradictions(auto_apply=True)` and asserts `applied >= 1`.
- `design-doc-policy-evidence-missing` — The docs search for `detected[-_]contradiction|auto[-_]accept|scrutiniz|cheap` returned zero matches, and `.claude/skills/reasons/SKILL.md` was not read.
