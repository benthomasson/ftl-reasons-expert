# Semantic Contradiction Detection

**Date:** 2026-05-10
**PR:** [#131](https://github.com/benthomasson/ftl-reasons/pull/131) (open)
**Builds on:** [#130](https://github.com/benthomasson/ftl-reasons/pull/130) (semantic dedup, merged)

## What Changed

The `contradictions` command gained a `--semantic` flag that clusters beliefs by embedding similarity before sending each cluster to the LLM for contradiction analysis. Previously, beliefs were sent in random batches of 50 -- topically related beliefs often ended up in different batches, reducing the chance of catching contradictions between them.

With `--semantic`, beliefs are first embedded via sentence-transformers (all-MiniLM-L6-v2) and clustered with KMeans via the existing `list_clusters()` infrastructure. Each cluster is then sent to the LLM as a batch, ensuring beliefs about the same subsystem are analyzed together.

## First Run Results

Running against the expert knowledge base (575 IN beliefs, 20 clusters) found **23 contradictions** -- 10 High, 12 Medium, 1 timeout (cluster 1 with 55 beliefs exceeded the 300s LLM timeout).

### Contradiction Clusters by Subsystem

The semantic grouping clearly worked -- contradictions cluster around specific subsystems rather than being scattered:

**check_stale mutation semantics (4 contradictions, 3 High):**
- `check-stale-is-read-only` vs `check-stale-and-hash-sources-mutate-in-place` -- direct negation about whether check_stale mutates the network
- `hash-sources-mutates-network` describes an "intentional asymmetry" that contradicts the mutation claim
- `staleness-is-conservative-ci-gate` claims "never mutates state" which conflicts with the mutation belief
- Root cause: beliefs were written at different times with different understandings of the `upgrade_hashes` parameter

**compact() budget semantics (2 contradictions, 1 High):**
- `compact-never-exceeds-budget` (hard guarantee) vs `compact-budget-is-soft-cap` / `compact-budget-soft-ceiling` (~25% overshoot allowed)
- `compact-section-order-conflict` -- IN/OUT/Nogoods vs Nogoods/OUT/IN ordering

**PgApi simulation status (3 contradictions, 2 High):**
- `pgapi-partial-api-coverage` says simulation is "deferred to future work"
- `pg-what-if-is-safely-simulated` and `both-backends-support-safe-hypothetical-reasoning` describe simulation as already implemented
- `pg-api-what-if-never-mutates` calls it "read-only" while the simulation belief describes "real mutations then rollback"

**Access control semantics (1 High):**
- `pg-access-control-raises-permission-error` uses intersection semantics (any overlap grants access)
- `visibility-is-subset-check`, `access-tags-subset-gate`, `visible-to-superset-semantics` all use subset semantics (all tags must match)

**Staleness detection scope (3 Medium):**
- `staleness-checking-is-comprehensive` ("detects all nodes") contradicted by `check-stale-skips-out-nodes` and `check-stale-requires-both-source-fields`
- `staleness-uses-full-sha256` ("no prefix matching") vs `truncated-hash-threshold-is-16-chars` (prefix comparison exists)

**Import/out-belief handling (2 Medium):**
- `import-agent-out-beliefs-get-empty-justifications` vs `import-agent-out-beliefs-not-resurrected` -- the "not resurrected" belief references antecedents that the "empty justifications" belief says don't exist
- `external-deps-become-premises` would make OUT beliefs IN as premises, conflicting with the empty-justifications belief

**LLM subprocess invocation (2 Medium):**
- `llm-prompt-always-via-stdin` ("always subprocess.run") vs `derive-uses-subprocess-not-sdk` (uses asyncio.create_subprocess_exec)
- `llm-subprocess-isolation-prevents-recursion` ("centrally enforced") vs `derive-strips-claudecode-env` / `ask-strips-claudecode-env` (each module strips independently)

**Derived belief soundness (1 Medium):**
- `derived-belief-soundness-is-llm-only` ("no post-hoc audit") vs `review-and-contradictions-catch-orthogonal-errors` (review catches invalid reasoning)

**Nogood ID management (1 Medium):**
- `nogood-id-collision-prevention` (unconditional set) vs `nogood-id-counter-is-monotonic` / `import-beliefs-nogood-id-uses-high-water-mark` (high-water mark, only increases)

**Mutation guarantee characterization (2 Medium):**
- `network-mutations-are-audited-and-index-consistent` (2 invariants) vs `mutations-are-atomic-audited-and-index-consistent` (3 guarantees)
- Disagreement on the third property: atomicity vs observability

**State reversibility (1 High):**
- `reasoning-engine-is-deterministic-and-reversible` ("every state transition is fully reversible") vs `challenge-destroys-premise-identity` / `dialectical-defeat-is-reversible-but-identity-is-permanent` (identity transformation is permanent and irreversible)

**Summary node hiding (1 High):**
- `compact-summary-nodes-hide-covered` (no condition on summary node status) vs `compact-summary-hiding-requires-in` (summary must be IN to hide)

**Batch size (1 High):**
- `list-negative-batches-at-40` vs `list-negative-batches-at-50` -- a single code constant cannot be both values

## Architecture

The implementation adds a `detect_contradictions_semantic()` function to `contradictions.py` that sits alongside the existing `detect_contradictions()`. The API function dispatches between them based on the `semantic` flag.

```
beliefs --> list_clusters() --> [cluster1, cluster2, ...] --> LLM per cluster --> parse NOGOODs
                                    |                              |
                            sentence-transformers          existing prompt/parse
                            + KMeans (from cluster.py)     pipeline (unchanged)
```

Single-belief clusters are skipped (no contradiction possible). Large clusters exceeding `CONTRADICTION_BATCH_SIZE` (50) are sub-batched within the cluster.

## Observations

1. **Semantic clustering catches what random batching misses.** The check_stale contradictions involve 4 beliefs about the same function -- with random batching of 575 beliefs into groups of 50, the probability of all 4 landing in the same batch is very low.

2. **Most contradictions are belief-age problems.** Beliefs written at different times capture different understandings of the same code. The check_stale beliefs likely reflect code changes where `upgrade_hashes` was added but earlier beliefs were not updated.

3. **Some contradictions are precision problems.** "Every state transition is fully reversible" is an over-generalization that one specific transition disproves. The belief is mostly true but the universal quantifier makes it contradictable.

4. **The timeout on cluster 1 (55 beliefs) suggests a need for the intermediate-save feature.** Long runs should checkpoint results so a timeout or interruption doesn't lose completed clusters.

## Next Steps

- Triage the 23 contradictions: verify against current code, retract stale beliefs, refine over-generalizations
- Add intermediate result saving to contradiction detection for long runs
- Consider re-running with `--timeout 600` to handle the cluster-1 timeout



High (10):
- check-stale-mutation-vs-read-only — check_stale described as both mutating and read-only
- check-stale-mutation-vs-asymmetry-claim — conflicting accounts of whether check_stale mutates
- check-stale-mutation-vs-ci-gate-immutability — same check_stale mutation contradiction
- compact-budget-hard-vs-soft — budget described as both hard guarantee and soft ~25% cap
- list-negative-batch-size-conflict — batch size claimed as both 40 and 50
- simulation-deferred-vs-implemented — PgApi simulation both "deferred" and "already working"
- both-backends-vs-simulation-deferred — same PgApi simulation contradiction
- summary-hiding-condition-conflict — whether OUT summary nodes hide covered nodes
- intersection-vs-subset-semantics — access control using intersection vs subset check
- reversibility-vs-permanent-identity — "every state transition reversible" vs "identity transformation is permanent"

Medium (12):
- out-belief-antecedent-mismatch — empty justifications vs referencing antecedents
- absent-deps-vs-out-belief-import — conflicting rules for OUT beliefs with absent deps
- soundness-validation-scope — soundness "LLM-only" vs review catching reasoning errors
- comprehensive-vs-skips-out-nodes — "detects all" vs skips OUT nodes
- comprehensive-vs-requires-both-fields — "detects all" vs skips nodes missing source_hash
- full-sha256-vs-prefix-matching — "no prefix matching" vs truncated hash prefix comparison
- what-if-readonly-vs-mutates — what_if "read-only" vs "performs real mutations then rollback"
- subprocess-api-mismatch — "always subprocess.run" vs asyncio.create_subprocess_exec
- claudecode-stripping-locus — "centrally enforced" vs each module strips independently
- compact-section-order-conflict — IN/OUT/Nogoods order vs Nogoods/OUT/IN order
- collision-prevention-vs-monotonicity — unconditional set vs high-water mark max()
- mutation-guarantee-count-mismatch / mutation-triple-identity-conflict — 2 vs 3 guarantees, atomicity vs observability 
