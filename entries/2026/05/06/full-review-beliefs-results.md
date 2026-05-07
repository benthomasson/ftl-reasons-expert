# Full `review-beliefs` Pass — Results and Analysis

## Overview

Ran `review-beliefs` across all 448 derived beliefs in the ftl-reasons-expert network. This is the first complete review pass using the command introduced in PR #97 (v0.33.0). The review evaluated each derivation on three axes: validity, sufficiency, and necessity.

## Execution

- **Total derived beliefs reviewed:** ~420 (of 448; some batches timed out)
- **LLM calls:** ~25 (batches of 20 beliefs each)
- **Timeout:** 300s initially, bumped to 600s after 2 batch timeouts
- **Sessions:** 8 review rounds over one day, with manual triage between rounds
- **Model:** Claude (via `invoke_model`)

## Aggregate Results

| Metric | Count |
|--------|-------|
| Beliefs reviewed | ~420 |
| Passed all axes | ~340 (81%) |
| Invalid | 44 retracted |
| Insufficient (not also invalid) | ~10 (triaged manually) |
| Unnecessary antecedents | ~20 (not acted on) |
| Cascade retractions | ~286 |
| Network before | 812 IN / 79 OUT |
| Network after | 526 IN / 365 OUT |
| Reduction | 35% of IN beliefs removed |

## Failure Taxonomy

The invalid derivations fell into recurring categories:

### 1. Scope Over-Generalization (most common)
Claims "all X" or "every X" when antecedents establish only specific instances. The leap from "these three modules do Y" to "all modules do Y" without proving the enumeration is exhaustive.

Examples:
- `all-network-modifications-are-auditable-and-topology-preserving` — covered mutations and dedup but not import
- `every-mutation-reports-its-effects` — only 3 of many mutation types addressed
- `output-governance-is-complete-authorized-and-ci-ready` — "primary outputs" generalized to "all outputs"

### 2. Conjunction Distribution Fallacy
System A has property P, system B has property Q, therefore the combined system has both P and Q at every point. Properties established for different subsystems get cross-applied without justification.

Examples:
- `all-output-is-deterministic-authorized-and-resilient` — combined properties from different path sets
- `revision-is-lifecycle-safe-and-semantics-preserving` — swapped properties between revision mechanisms
- `system-output-is-complete-bounded-and-ci-ready` — token budgets from compact, exit codes from staleness

### 3. Strength Escalation
Antecedents establish a moderate property ("controlled", "bounded", "defensive") but the conclusion claims an absolute ("never leaks", "zero gaps", "no hidden fragility").

Examples:
- `external-integration-is-hardened-and-boundary-controlled` — "controlled" doesn't entail zero-leakage
- `architecture-has-no-hidden-fragility` — can't prove a universal negative from two positive properties
- `complete-system-is-self-correcting` — "no inconsistency persists" broader than contradiction + staleness

### 4. Causal Overclaim
Two properties are observed together and the derivation asserts one causes the other, when the antecedents only establish co-presence.

Examples:
- `minimality-produces-uniformity-and-determinism` — co-presence claimed as causation
- `resource-sustainable-lifecycle-has-no-gaps` — gaplessness attributed to resource sustainability without evidence

### 5. Missing Bridge Assumptions
Two individually sound antecedents require an unstated connector to reach the conclusion.

Examples:
- `agent-beliefs-undergo-full-revision` — needed bridge between "self-contained" agent subsystem and general revision
- `full-user-stack-is-verified-atomic-delegation` — atomicity unstated, only traceability established

## Cascade Analysis

Retracting 44 beliefs caused 286 additional beliefs to go OUT — a 6.5x amplification factor. This confirms the pattern that invalid derivations at depth 8-15 are load-bearing for many higher-level conclusions.

The largest single cascade: retracting `agent-beliefs-undergo-full-revision` took 52 dependents with it, mostly "origin-agnostic" and "invariant-preserving" claims that all traced back to the same missing bridge.

As the review progressed, cascades shrank because earlier retractions had already removed the most dependent over-generalizations.

## What Survived

The 526 remaining IN beliefs are:
- **All premises** (direct code observations) — untouched
- **Well-grounded derived beliefs** at depth 1-4 — individual reasoning steps that don't over-generalize
- **Properly scoped higher-level beliefs** — conclusions that match the specificity of their antecedents

The review did not find issues with:
- Premise accuracy (premises are code observations, not derivations)
- Single-step derivations at low depth
- Beliefs with GATE conditions (outlist guards)
- Formatting or structural issues in the review pipeline itself

## Observations on the Review Process

**Direct-antecedent-only review is correct.** The reviewer sees only immediate antecedents, which occasionally produces false positives on sufficiency (support exists one level deeper). We triaged these manually and found the false positive rate acceptable. Adding transitive context would bloat prompts without proportional benefit.

**Deep chains accumulate error.** Each derivation step introduces a small generalization. By depth 10+, conclusions have drifted far from what premises support. This is analogous to floating-point error accumulation — individually reasonable rounding at each step, catastrophically wrong at the end.

**The `--min-depth` flag is high-value.** Deep beliefs are higher risk and higher leverage per LLM call. Future reviews should prioritize `--min-depth 8`.

**Timeout of 600s is needed.** The default 300s caused 2 batch failures. Batches with complex justification chains take longer for the LLM to evaluate.

**Unnecessary antecedents are common but harmless.** ~20 beliefs had redundant antecedents. These make justifications non-minimal but don't affect validity. Not worth the churn of fixing unless a cleanup pass is specifically desired.

## Implications for `derive --exhaust`

The derivation engine produced these invalid beliefs because it:
1. Generates plausible-sounding conclusions from antecedent texts
2. Validates structurally (antecedents exist, no cycles, Jaccard dedup) but not semantically
3. Tends toward increasingly general claims at higher depths

`review-beliefs` closes this gap — it's the semantic validation layer that `derive` lacks. The recommended workflow is now:

```
derive --exhaust → review-beliefs → retract invalid → derive --exhaust → ...
```

This cycle should converge: each pass produces fewer but better-grounded beliefs, and the review catches over-generalizations before they become load-bearing.

## Next Steps

- Run `derive --exhaust` to see if the engine can re-derive retracted beliefs with sounder justifications
- Compare re-derived beliefs against the retracted set to measure improvement
- Consider adding `--auto-retract` for future review passes once confidence in the reviewer is established
- Track whether the derive→review→retract cycle converges to a stable belief set
