# Belief Staleness in Business Planning Networks

## The Problem

TMS belief networks sourced from documents face a fundamental staleness problem. Documents go out of date soon after they're written — org charts change, strategies pivot, market conditions shift, projects complete or get cancelled. Beliefs extracted from those documents inherit this staleness but carry no expiration date.

In a code-expert network, staleness is cheap to detect and cheap to verify:
- **Detection**: `check-stale` hashes source files and flags beliefs whose source material changed on disk. Automated, instant.
- **Verification**: Read the updated code, decide if the belief still holds. Minutes per belief.

In a business planning network, both steps are expensive:
- **Detection**: Documents live in Google Drive, not on disk. No hash-based change detection. Even if we could detect document changes, the document might have changed in ways that don't affect the belief, or the belief might be stale even though the document hasn't changed (the document itself is outdated).
- **Verification**: Confirming whether a business premise still holds may require scheduling meetings, pulling financial reports, checking with stakeholders, or waiting for outcomes. Hours to weeks per belief.

## Two Types of Staleness

### 1. Source Staleness
The document a belief was extracted from has been updated or replaced. The belief may or may not still hold — it needs re-examination.

Example: A belief extracted from a Q3 strategy doc. Q4 strategy has since been published. The belief's source is stale even if the conclusion might still be valid.

### 2. World Staleness
The belief was accurate when proposed but the real world has changed. No document was updated — the premise simply isn't true anymore.

Example: "Team X has 12 engineers" was true when extracted from the org chart. Three people have since left. No document was updated to reflect this.

World staleness is harder because there's no artifact to trigger re-examination. The belief sits in the network looking valid until someone happens to notice it's wrong.

## Why This Matters

Stale premises are load-bearing. A single stale premise deep in the network can support dozens of derived conclusions. If "Team X has 12 engineers" is wrong, then derived beliefs about team capacity, project timelines, and resource allocation are all suspect — but they look fine because their justification chains are structurally valid.

This is the same error accumulation pattern we saw with `review-beliefs` (over-generalization at depth 8-15), but the failure mode is different:
- **review-beliefs** catches bad reasoning from valid premises
- **Staleness** catches valid reasoning from outdated premises

Both produce conclusions that look justified but are wrong.

## Current Tools and Gaps

| Tool | Code expert | Business expert |
|------|------------|-----------------|
| Source change detection | `check-stale` (hash-based) | No equivalent for Google Drive |
| Premise verification | Read source code (minutes) | Stakeholder conversations (hours-weeks) |
| Review cadence | On every PR / derive cycle | Manual, ad-hoc |
| Staleness signal | File hash mismatch | None — silent decay |

## Ideas Worth Exploring

### Prioritized Review by Impact
The TMS has the dependency graph to compute which stale beliefs matter most. Rank candidates by:
- **Dependent count**: How many beliefs depend on this premise?
- **Cascade size**: If retracted, how many beliefs go OUT? (`what-if retract`)
- **Depth reach**: How deep into the derivation tree does this premise's influence extend?

Expensive verification should target high-impact premises first. A stale belief with 50 dependents is worth a meeting to verify. A stale leaf with no dependents can wait.

### Time-Based Staleness
Tag beliefs with a "reviewed" timestamp. Flag beliefs that haven't been verified in N days/weeks. This doesn't detect actual staleness but ensures periodic re-examination.

Could be as simple as metadata: `last_reviewed: 2026-05-06`. Then `reasons list --stale-since 30d` surfaces beliefs not reviewed in 30 days, ordered by impact.

### Source Document Tracking
For Google Drive-sourced beliefs, track the document ID and last-modified timestamp. Periodically check whether the source doc has been modified since the belief was extracted. This is the `check-stale` pattern adapted for remote documents.

Requires Google Drive API integration — heavier than file hashing but feasible.

### Review Batching
Instead of reviewing one belief at a time, batch related premises for review in a single meeting or review session. "Here are 15 premises about Team X's capacity — are these still accurate?" is more efficient than 15 separate inquiries.

The clustering approach (issue #111) could group beliefs by topic for efficient batch review, not just for derivation.

## The Core Tension

The fundamental tension is verification cost vs. network reliability. In a code network, we can afford to verify every premise because verification is cheap. In a business network, we have to be strategic about what we verify because verification is expensive.

The TMS should help manage this tension by:
1. Surfacing which beliefs are most likely stale (age, source changes)
2. Ranking them by impact (dependency graph)
3. Batching them for efficient review (clustering)
4. Recording the outcome (retract, update text, confirm still valid)

This is an unsolved problem worth iterating on as the redhat-expert network grows.
