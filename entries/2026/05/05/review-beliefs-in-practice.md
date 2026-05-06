# `review-beliefs` in Practice — First Results

## Background

The `review-beliefs` command (PR #97, merged in v0.33.0) sends derived beliefs back to an LLM for semantic validation. Each belief is presented with its direct antecedent texts, and the reviewer evaluates three axes: **validity** (does the conclusion follow?), **sufficiency** (are the antecedents enough?), and **necessity** (are all antecedents load-bearing?).

This entry documents the first live run against the ftl-reasons-expert belief network (891 total beliefs, 448 derived at time of review).

## What We Did

Reviewed 80 derived beliefs in 4 batches of 20 (4 LLM calls total). Each batch sends all 20 beliefs in a single prompt and receives a JSON array of results.

### Batch Results

| Batch | Passed | Invalid | Insufficient | Unnecessary |
|-------|--------|---------|--------------|-------------|
| 1     | 17     | 2       | 2            | 1           |
| 2     | 13     | 5       | 6            | 1           |
| 3     | 18     | 1       | 1            | 1           |
| 4     | 17     | 1       | 2            | 1           |

**Totals across 80 beliefs:** 65 passed (81%), 9 invalid, 11 insufficient (overlap — most invalid beliefs were also insufficient), 4 unnecessary antecedents.

### Actions Taken

- **8 beliefs retracted** as invalid derivations. These cascaded OUT a total of ~100 dependent beliefs, bringing the network from 812 IN to 710 IN.
- **2 beliefs fixed** by adding missing antecedents that existed in the network but weren't wired into the justification.
- **1 insufficient belief left as-is** after determining the missing link existed one level deeper in the transitive antecedent chain (a false positive).
- **4 unnecessary antecedents identified** but not acted on (valid but not minimal — harmless).

## What the Reviewer Catches

The reviewer is strongest at detecting **logical leaps in derived beliefs**, particularly:

1. **Scope over-generalization**: Claiming "all X" when antecedents only establish "these specific X". Example: `all-output-is-deterministic-authorized-and-resilient` combined properties from different path sets into a universal claim.

2. **Conjunction distribution fallacy**: Assuming that because system A has property P and system B has property Q, the combined system has both P and Q simultaneously at every point. Example: `all-transformations-are-deterministic-traceable-and-boundary-safe` cross-attributed properties between transformation types.

3. **Missing bridge assumptions**: Two antecedents that are individually sound but require an unstated assumption to connect them. Example: `agent-beliefs-undergo-full-revision` needed an unstated bridge between the agent subsystem being "self-contained" and beliefs participating in general revision.

4. **Strength escalation**: Antecedents establish a moderate property ("controlled", "bounded") but the conclusion claims an absolute ("never leaks", "zero gaps"). Example: `external-integration-is-hardened-and-boundary-controlled`.

## What the Reviewer Misses

The reviewer only sees **direct antecedents**, not the transitive chain. This produces false positives on sufficiency when the missing support exists one level deeper. Example: `source-integrity-spans-hashing-through-detection` was flagged as insufficient because the reviewer couldn't see that `staleness-uses-full-sha256` (establishing the hash-consumption link) was an antecedent of the antecedent.

**Decision**: Keep reviewing steps in isolation. This is the correct granularity for logical validity. Triage sufficiency flags manually by checking transitive support before acting.

## Pattern: Deep Derivation Chains Accumulate Error

The most problematic beliefs were at depth 8-15 — the apex of long derivation chains. Each step introduces a small generalization, and by the time you reach the top, the conclusion has drifted far from what the premises actually support. This is analogous to error accumulation in floating-point arithmetic: individually reasonable rounding at each step, but the final result is wrong.

The retraction cascades confirm this: retracting one invalid belief at depth 8 caused 52 dependents to go OUT, because everything above it was built on the same shaky foundation.

**Implication**: The `--min-depth` flag is useful for targeting review effort. Deep beliefs are higher risk and higher leverage — reviewing them first catches more issues per LLM call.

## Cost and Throughput

- **4 LLM calls** to review 80 beliefs (20 per batch)
- Each call takes ~30-60 seconds
- Reviewing all 448 derived beliefs would take ~23 calls
- The review found actionable issues in 19% of beliefs reviewed (15/80)
- After retraction cascades, the network shrank by 12.5% (812 → 710 IN)

## Network State After Review

- **710 IN / 181 OUT** (891 total)
- 8 beliefs retracted directly, ~100 cascaded OUT
- 2 beliefs strengthened with additional antecedents
- Core premises and well-grounded derived beliefs unaffected

## Next Steps

- Review remaining ~268 derived IN beliefs
- Consider prioritizing `--min-depth 8` to target the highest-risk derivations
- After full review, run `derive --exhaust` to see if the engine can re-derive retracted beliefs with sounder justifications
- Track false positive rate on sufficiency to validate the direct-antecedent-only approach
