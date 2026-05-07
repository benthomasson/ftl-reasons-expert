# First `contradictions` Run — Results and Analysis

## Overview

Ran the new `reasons contradictions` command (PR #99, v0.34.0) across all 526 IN beliefs in the ftl-reasons-expert network. This is the first live run of LLM-powered contradiction detection — sending batches of beliefs to an LLM to find sets that cannot all be true simultaneously.

## Execution

- **Beliefs checked:** 526 (all IN)
- **Batches:** 11 (50 beliefs each, randomly shuffled)
- **Contradictions found:** 8
- **Mode:** `--dry-run` (no auto-apply)

## Findings

### High Severity (4)

| Nogood ID | Claims | Resolution |
|-----------|--------|------------|
| network-sole-vs-pg-reimplements | `network-is-sole-truth-propagation-engine`, `pg-reimplements-network-in-sql` | Retracted `network-is-sole-truth-propagation-engine` — PgApi has its own SQL propagation |
| pg-dialectics-implemented-vs-deferred | `pg-reimplements-network-in-sql`, `pgapi-partial-api-coverage` | Retracted `pg-reimplements-network-in-sql` — over-generalizes, claims dialectics are reimplemented when they're deferred |
| derive-llm-call-contradiction | `derive-no-llm-call`, `derive-strips-claudecode-env` | Retracted `derive-no-llm-call` — `_derive_one_round` does spawn the model subprocess |
| compact-budget-hard-vs-soft | `compact-budget-guarantee`, `compact-budget-is-soft-cap`, `compact-budget-soft-ceiling` | Retracted `compact-budget-guarantee` — budget is a soft cap, can exceed by ~25% |

### Medium Severity (4)

| Nogood ID | Claims | Resolution |
|-----------|--------|------------|
| staleness-comprehensive-vs-silent-skip | `staleness-checking-is-comprehensive`, `check-stale-requires-both-source-fields`, `source-lifecycle-is-fail-safe-and-gapless` | Filed issue #100 — nodes with partial source metadata silently skipped |
| check-stale-readonly-vs-mutates | `check-stale-report-only`, `check-stale-and-hash-sources-mutate-in-place` | Filed issue #101 — `upgrade_hashes=True` mode mutates, should be a separate command |
| ask-fault-tolerance-vs-uncaught-exception | `ask-is-fault-tolerant-and-bounded`, `invoke-claude-raises-when-binary-missing` | Filed issue #102 — `FileNotFoundError` not caught by `ask()` |
| pgapi-commit-vs-whatif-rollback | `pgapi-one-transaction-per-method`, `pg-what-if-is-safely-simulated` | Fixed text: "commits" → "commits or rolls back" via `reasons update` |

## Actions Taken

- **4 beliefs retracted** (all high-severity direct contradictions)
- **3 issues filed** (#100, #101, #102) for medium-severity contradictions that represent real code gaps
- **1 belief text updated** to resolve an imprecise premise
- **0 cascades** — all retracted beliefs were leaves or had already-OUT dependents

## Contradiction Categories

The contradictions fell into two categories:

### 1. Absolute Claims vs. Exceptions
Most contradictions involved a belief making an absolute claim ("all", "never", "sole", "every") when another belief documented a specific exception. Examples:
- "sole truth propagation engine" vs. PgApi's SQL propagation
- "never calls an LLM" vs. spawning the model subprocess
- "never exceeds budget" vs. soft cap with ~25% overflow
- "always returns a string" vs. uncaught `FileNotFoundError`

### 2. Imprecise Premises
One contradiction was an imprecise premise rather than a wrong one — `pgapi-one-transaction-per-method` said "commits" when it should have said "commits or rolls back." This was fixable with a text update rather than retraction.

## Comparison with `review-beliefs`

| Aspect | `review-beliefs` | `contradictions` |
|--------|------------------|-------------------|
| What it checks | Individual derivation steps | Beliefs against each other |
| Scope | Derived beliefs only | All IN beliefs |
| Failure mode | Over-generalization in reasoning | Incompatible claims across premises |
| Typical fix | Retract invalid derivation | Retract one claim, file issue, or fix text |
| Cascade impact | High (deep chains) | Low (contradictions often at leaf level) |
| Batch size | 20 (includes antecedent context) | 50 (one-liner per belief) |

`review-beliefs` catches bad reasoning; `contradictions` catches inconsistent facts. They complement each other — review-beliefs already removed 44 over-generalizations, and contradictions found 8 remaining inconsistencies among the surviving beliefs.

## Process Observations

**Random shuffling is essential.** Beliefs are shuffled before batching so each run produces different groupings. A contradiction between belief #1 and belief #51 would never be found with sequential batching. Over multiple runs, shuffling probabilistically covers more cross-belief pairs.

**Medium-severity contradictions are often real code issues.** The 4 medium-severity findings all pointed to genuine gaps in the codebase — missing exception handling, mutation in a supposedly read-only function, silent skipping of eligible nodes. These were filed as issues rather than resolved purely within the belief network.

**The `update` command was needed immediately.** While resolving contradictions, we hit a case where a premise was imprecise but not wrong. There was no way to fix the text without superseding or direct SQL. Filed issue #103 and implemented `reasons update` (PR #104) during the same session.

## Network State After Contradictions

- **522 IN / 369 OUT** (891 total)
- 4 beliefs retracted directly, 0 cascaded
- 1 belief text corrected
- 3 issues filed for code-level fixes

## Recommended Workflow

The full belief maintenance cycle is now:

```
derive --exhaust → review-beliefs → retract invalid → contradictions → retract/fix/file → derive --exhaust → ...
```

Each tool catches a different class of error:
- `derive` generates new beliefs
- `review-beliefs` validates individual reasoning steps
- `contradictions` finds cross-belief inconsistencies
- Manual triage resolves what automation surfaces
