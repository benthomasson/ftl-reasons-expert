---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Determine whether `basis` (`source-divergence` vs `prior-knowledge`) or `failure_mode` controls auto-accept vs manual review; only prompt text hints at this.

## What the code says (and what it hints at)

### Short answer

The **prompt text** hints that `basis` — specifically `source-divergence` vs `prior-knowledge` — is the intended auto-accept vs manual-review axis. `failure_mode` is purely a defect taxonomy with no accept/review annotation. **However, in the code actually read, no auto-accept path exists at all**: every proposal becomes a markdown entry with a hand-run command, regardless of `basis` or `failure_mode`.

### Evidence from the prompt

`PROPOSE_UPDATE_PROMPT` in `reasons/propose_update.py` (around the `basis` enumeration) contrasts the three `basis` values in explicitly trust-shaped language:

- `source-divergence`: *"the source file/URL changed (mechanically verifiable, cheap to accept)"*
- `detected-contradiction`: *"conflicts with another belief (nogood)"*
- `prior-knowledge`: *"your own assessment (needs scrutiny — reviewer may share your blind spot)"*

Only `basis` carries these "cheap to accept" / "needs scrutiny" annotations. That wording is the only signal anywhere in the observation set that one class of proposal should be rubber-stamped and another should be human-gated.

By contrast, the `failure_mode` enumeration in the same prompt is purely descriptive:
- `smuggled-premise`, `unsupported-superlative`, `false-causal-link`, `domain-conflation`, `contradicted-by-source`, `stale`.

None of these are annotated with an accept/review consequence — `failure_mode` classifies *what is wrong*, not *who signs off*. So, **as a design intent expressed in the prompt, `basis` is the review-tier switch, not `failure_mode`.**

Note that `detected-contradiction` sits in the middle with no explicit hint. The prompt granularity is really "mechanically verifiable vs. LLM self-assessment," and `detected-contradiction` is closer to mechanically verifiable (a nogood exists), but that's inference, not stated.

### What the code actually does

The implementation in `reasons/propose_update.py` does **not** branch on `basis` (or on `failure_mode`) at all:

1. `parse_update_proposals` normalizes both fields and defaults an unknown `basis` to `prior-knowledge` — no routing.
2. `format_proposal_markdown` renders `Failure mode:` and `Basis:` as **labels**, then emits a suggested shell command:
   - `reasons update <id> "<text>"` for updates
   - `reasons retract <id> --reason "<failure_mode>: <comment>"` for retracts
3. `format_proposals_file` writes them all into one document and states: *"Review each proposal below. Accept by running the suggested command."*

So today, every proposal is **manual**, uniformly. Nothing in `propose_updates` returns an "auto-accepted" subset, and nothing filters proposals by `basis == "source-divergence"`.

### Caveat about the locate results

The observations show three locate searches returning `total_count: 0`:

- `source-divergence|prior-knowledge`
- `failure_mode`
- `auto[_-]?accept|manual[_-]?review|auto[ -]?accept|manual[ -]?review`

The first two **cannot both be right** with `read_propose_update` in the same snapshot: `source-divergence`, `prior-knowledge`, and `failure_mode` appear as hardcoded literals and dict keys inside `reasons/propose_update.py` (in `FAILURE_MODES`, `BASES`, `parse_update_proposals`, and `format_proposal_markdown`). The only consistent interpretations are:

- the locate scope is a different directory/file set than `reasons/propose_update.py`, or
- the locate ran against a different revision than the reads.

Either way, the negative result for `auto_accept|manual_review` **cannot be taken as authoritative proof of absence** — it is only suggestive. The read of `propose_update.py` is the stronger evidence: it contains no `auto_accept`/`manual_review` branch.

### Bottom line

- **If you rely only on the prompt**: `basis` (`source-divergence` → auto-accept-ish, `prior-knowledge` → manual review) is the intended axis; `failure_mode` is orthogonal and descriptive.
- **If you rely on the actual implementation**: neither field controls anything; there is no auto-accept code path in the observed files, only manual-review markdown generation.
- **What's missing to close the loop**: the CLI entrypoint that calls `propose_updates` (e.g. a `cmd.py` / `cli.py` subcommand) — that is where a real `basis`-driven auto-accept gate would have to live, and it was not included in the observations.

## Topics to Explore

- [file] `reasons/propose_update.py` — Contains both the prompt wording that hints at `basis`-driven review tiers and the current all-manual rendering; the single source of truth for how proposals flow to humans.
- [function] `reasons/propose_update.py:format_proposal_markdown` — Renders `Failure mode:` / `Basis:` labels and builds `reasons update` / `reasons retract` shell commands; the place where a per-`basis` split would be visible if it existed.
- [general] `propose-update-cli-entrypoint` — Find the CLI command that invokes `propose_updates`; a `basis`-based auto-accept gate would have to live here (e.g. skip markdown for `source-divergence`, keep it for `prior-knowledge`).
- [general] `locate-scope-discrepancy` — The locate searches returned 0 matches for strings that are clearly literals in `reasons/propose_update.py`; verify the search scope/revision to know whether the negative result means "no auto-accept exists" or "tool didn't scan this file."
- [general] `detected-contradiction-tier` — The prompt does not annotate `detected-contradiction` as cheap or scrutinized; check any design docs or downstream nogood handling to see whether it was intended for auto-accept or review.

## Beliefs

- `basis-controls-review-tier-in-prompt` — `PROPOSE_UPDATE_PROMPT` annotates only `basis` values with accept/review language ("cheap to accept" for `source-divergence`, "needs scrutiny" for `prior-knowledge`), making `basis` the intended auto-accept/manual-review axis.
- `failure-mode-is-descriptive-taxonomy` — `failure_mode` values in `FAILURE_MODES` and the prompt classify defect types with no accept/review policy annotation anywhere.
- `propose-updates-emits-markdown-not-decisions` — `propose_updates` + `format_proposals_file` produce a markdown document of proposals with suggested commands; they do not return or apply an auto-accepted subset.
- `all-proposals-require-manual-command` — `format_proposals_file` unconditionally directs the reader to "Accept by running the suggested command," with no `basis`-based branching.
- `unknown-basis-defaults-to-prior-knowledge` — `parse_update_proposals` coerces an unrecognized `basis` value to `prior-knowledge`, defaulting everything to the "needs scrutiny" tier rather than auto-accept.
