# derive

[Back to index](index.md)

### cluster-derive-is-semantically-informed-and-deterministic
**Status:** IN

When using cluster-based belief selection, the derive pipeline achieves semantically-informed budget allocation (embedding-based grouping ensures topical diversity across the prompt) with end-to-end determinism (sorted embedding order, fixed-seed clustering, and exact budget counts feed into reproducible prompt construction with accurate token allocation).

**Depends on:** [cluster-selection-is-deterministic-and-budget-exact](deterministic.md#cluster-selection-is-deterministic-and-budget-exact), [derive-prompt-is-deterministic-and-reproducible](derive.md#derive-prompt-is-deterministic-and-reproducible)

### derive-achieves-flexibility-with-reproducibility
**Status:** IN

The derive pipeline resolves the tension between strategic flexibility and deterministic reproducibility: three budget strategies (alphabetical truncation, random sampling, semantic clustering) provide diverse exploration approaches, while fixed-seed deterministic sampling and accurate proportional allocation ensure each strategy produces identical results across runs.

**Depends on:** [derive-budget-is-flexible-and-efficient](derive.md#derive-budget-is-flexible-and-efficient), [derive-prompt-is-deterministic-and-reproducible](derive.md#derive-prompt-is-deterministic-and-reproducible)

### derive-agent-budget-proportional
**Status:** IN

When agents are present, `_build_beliefs_section` allocates prompt token budget proportionally to each agent's belief count, with a floor of 5 beliefs per agent

**Supports:** [derive-budget-allocation-is-accurate](derive.md#derive-budget-allocation-is-accurate)

### derive-agent-count-bug
**Status:** OUT

`_build_beliefs_section` has a bug: `count += len(belief_ids)` is inside the per-belief loop instead of outside it, inflating the count and shrinking the non-agent budget below intended size

**Supports:** [all-external-inputs-produce-correct-state](external.md#all-external-inputs-produce-correct-state), [all-external-inputs-safely-integrated](external.md#all-external-inputs-safely-integrated), [belief-currency-is-actively-managed](other.md#belief-currency-is-actively-managed), [complete-operational-uniformity-across-all-sources](complete.md#complete-operational-uniformity-across-all-sources), [complete-system-is-self-correcting](complete.md#complete-system-is-self-correcting), [complete-unified-system-is-production-ready](complete.md#complete-unified-system-is-production-ready), [derive-budget-allocation-is-accurate](derive.md#derive-budget-allocation-is-accurate), [derive-pipeline-is-production-ready](derive.md#derive-pipeline-is-production-ready), [external-beliefs-are-safe-and-current](external.md#external-beliefs-are-safe-and-current), [external-integration-is-architecturally-safe](external.md#external-integration-is-architecturally-safe), [external-integration-preserves-all-invariants](external.md#external-integration-preserves-all-invariants), [self-correction-is-resource-sustainable](self.md#self-correction-is-resource-sustainable), [system-achieves-full-correctness](system.md#system-achieves-full-correctness), [unified-system-is-a-closed-self-maintaining-architecture](system.md#unified-system-is-a-closed-self-maintaining-architecture), [verified-mutation-correctness-across-boundaries](other.md#verified-mutation-correctness-across-boundaries)

### derive-apply-is-isolated-and-caller-validated
**Status:** IN

The derive apply stage achieves defense-in-depth: callers must run validation before apply (trust boundary), and even if invalid proposals reach apply, each proposal is wrapped in independent error handling so one failure cannot corrupt the batch.

**Depends on:** [derive-apply-isolates-per-proposal-errors](derive.md#derive-apply-isolates-per-proposal-errors), [derive-validate-before-apply](derive.md#derive-validate-before-apply)
**Supports:** [derive-pipeline-is-reproducible-and-defense-in-depth](derive.md#derive-pipeline-is-reproducible-and-defense-in-depth)

### derive-apply-isolates-per-proposal-errors
**Status:** IN

`apply_proposals` wraps each `api.add_node()` call in try/except and accumulates `(proposal, error_string)` tuples, so one malformed proposal does not abort the batch.

**Supports:** [batch-fault-isolation-is-universal-across-llm-operations](llm.md#batch-fault-isolation-is-universal-across-llm-operations), [derive-apply-is-isolated-and-caller-validated](derive.md#derive-apply-is-isolated-and-caller-validated)

### derive-budget-allocation-is-accurate
**Status:** IN

The derive pipeline's proportional belief-budget allocation produces correct per-agent token counts

**Depends on:** [derive-agent-budget-proportional](derive.md#derive-agent-budget-proportional), [derive-prompt-roundtrips-through-parser](derive.md#derive-prompt-roundtrips-through-parser)
**Supports:** [derive-prompt-is-deterministic-and-reproducible](derive.md#derive-prompt-is-deterministic-and-reproducible), [token-budgets-are-accurate-bidirectionally](other.md#token-budgets-are-accurate-bidirectionally)

### derive-budget-count-is-linear
**Status:** IN

Agent belief counting in `_build_beliefs_section` accumulates as N (number of beliefs shown per agent), not N² — the corrected behavior after the issue #23 fix where `count += len(belief_ids)` was moved outside the per-belief loop

**Supports:** [derive-budget-is-efficient-and-floor-bounded](derive.md#derive-budget-is-efficient-and-floor-bounded)

### derive-budget-floor-five
**Status:** IN

Each agent group and the non-agent group are guaranteed at least 5 belief slots in the prompt regardless of proportional budget allocation.

**Supports:** [derive-budget-is-efficient-and-floor-bounded](derive.md#derive-budget-is-efficient-and-floor-bounded)

### derive-budget-is-efficient-and-floor-bounded
**Status:** IN

The derive pipeline's per-agent budget allocation is both computationally efficient (O(N) linear accumulation, not quadratic) and representation-safe (each agent and local group guaranteed at least 5 belief slots), ensuring proportional allocation never starves minority agents.

**Depends on:** [derive-budget-count-is-linear](derive.md#derive-budget-count-is-linear), [derive-budget-floor-five](derive.md#derive-budget-floor-five)
**Supports:** [budget-enforcement-is-efficient-across-pipeline](other.md#budget-enforcement-is-efficient-across-pipeline), [derive-budget-is-flexible-and-efficient](derive.md#derive-budget-is-flexible-and-efficient), [derive-pipeline-is-safe-complete-and-efficient](derive.md#derive-pipeline-is-safe-complete-and-efficient)

### derive-budget-is-flexible-and-efficient
**Status:** IN

The derive pipeline's budget allocation is both strategically flexible (three selection strategies: alphabetical truncation, random sampling, semantic clustering) and computationally efficient (linear accumulation with guaranteed floor of 5 per agent group), enabling callers to trade off reproducibility, diversity, and semantic coherence without performance penalty.

**Depends on:** [derive-budget-is-efficient-and-floor-bounded](derive.md#derive-budget-is-efficient-and-floor-bounded), [derive-budget-three-strategies](derive.md#derive-budget-three-strategies)
**Supports:** [derive-achieves-flexibility-with-reproducibility](derive.md#derive-achieves-flexibility-with-reproducibility)

### derive-budget-local-floor-is-five
**Status:** IN

`_build_beliefs_section` guarantees at least 5 local beliefs are shown regardless of agent budget pressure, via `max(5, max_beliefs - count)`


### derive-budget-tests-parse-output-headers
**Status:** IN

Test implementation detail (regex parsing of `"showing N"` headers), not a production code invariant


### derive-budget-three-strategies
**Status:** IN

`_build_beliefs_section` supports three budget strategies — alphabetical truncation (default), random sampling (`sample=True`), and semantic clustering (`cluster=True`) — all controlled by a single `max_beliefs` parameter.

**Supports:** [derive-budget-is-flexible-and-efficient](derive.md#derive-budget-is-flexible-and-efficient)

### derive-depth-cycle-guard
**Status:** IN

`_get_depth` sets `memo[node_id] = 0` before recursing to prevent infinite recursion on cyclic justification chains; cycles resolve to depth 0

**Supports:** [derive-pipeline-is-exhaustive-and-terminating](derive.md#derive-pipeline-is-exhaustive-and-terminating), [propagation-is-crash-free](other.md#propagation-is-crash-free)

### derive-fail-soft-validation
**Status:** IN

`validate_proposals` filters invalid proposals into a skipped list rather than raising; `apply_proposals` catches per-item exceptions so one bad proposal never blocks others

**Supports:** [derive-pipeline-is-defensive](derive.md#derive-pipeline-is-defensive), [llm-integration-fails-softly-across-modules](llm.md#llm-integration-fails-softly-across-modules)

### derive-min-antecedents-is-prompt-only
**Status:** IN

The minimum-2-antecedents rule for derived beliefs is enforced only by the LLM prompt instructions, not validated in code by `validate_proposals`.

**Supports:** [complete-quality-lifecycle-spans-creation-through-egress](complete.md#complete-quality-lifecycle-spans-creation-through-egress), [derive-pipeline-has-end-to-end-quality-enforcement](derive.md#derive-pipeline-has-end-to-end-quality-enforcement), [derive-quality-is-comprehensively-code-enforced](derive.md#derive-quality-is-comprehensively-code-enforced), [llm-belief-pipeline-is-fully-quality-enforced](llm.md#llm-belief-pipeline-is-fully-quality-enforced), [review-driven-quality-lifecycle-is-fully-code-enforced](lifecycle.md#review-driven-quality-lifecycle-is-fully-code-enforced), [revision-has-code-enforced-derivation-constraints](revision.md#revision-has-code-enforced-derivation-constraints)

### derive-no-llm-call
**Status:** OUT

`derive.py` builds prompts and parses responses but never calls an LLM itself; the caller (CLI or API) is responsible for the model invocation.


### derive-parse-supports-two-format-versions
**Status:** IN

`parse_proposals` tries the new `### DERIVE id` format first, falling back to the older `### DERIVE: \`id\`` format only when the new parser returns zero matches.


### derive-parse-supports-two-formats
**Status:** IN

`parse_proposals` tries the new format first (v0.10+: `### DERIVE id`), falls back to old format (v0.9: `### DERIVE: \`id\``) only when no new-format matches are found

**Supports:** [system-tolerates-evolution-at-all-boundaries](system.md#system-tolerates-evolution-at-all-boundaries)

### derive-pipeline-achieves-end-to-end-fault-tolerance
**Status:** IN

The derive pipeline achieves end-to-end fault tolerance through three independent layers: proactive defense (fail-soft validation, Jaccard retraction guards, environment isolation), reactive resilience (partial results persisted via JSON reports after each round, error states signaled through return codes), and prompt reproducibility (deterministic sampling with fixed seeds enables consistent re-runs after failures).

**Depends on:** [derive-pipeline-is-defensive](derive.md#derive-pipeline-is-defensive), [derive-prompt-is-deterministic-and-reproducible](derive.md#derive-prompt-is-deterministic-and-reproducible), [derive-resilience-preserves-progress-on-error](derive.md#derive-resilience-preserves-progress-on-error)

### derive-pipeline-has-complete-coverage
**Status:** OUT

The derive pipeline achieves complete coverage along three axes: safety (fail-soft validation, Jaccard retraction guards, environment isolation), completeness (exhaustive exploration with guaranteed termination), and production-readiness (accurate proportional budgets, roundtrippable prompt format).

**Depends on:** [derive-pipeline-is-production-ready](derive.md#derive-pipeline-is-production-ready), [derive-pipeline-is-safe-and-complete](derive.md#derive-pipeline-is-safe-and-complete)
**Supports:** [all-llm-operations-achieve-coverage-and-fault-tolerance](llm.md#all-llm-operations-achieve-coverage-and-fault-tolerance), [llm-belief-pipeline-is-fully-quality-enforced](llm.md#llm-belief-pipeline-is-fully-quality-enforced)

### derive-pipeline-has-end-to-end-quality-enforcement
**Status:** OUT

The derive pipeline achieves end-to-end quality enforcement: defensive validation prevents invalid proposals, Jaccard retraction guards prevent re-derivation of known-bad conclusions, budget allocation is accurate, AND the minimum-antecedents rule for derived beliefs is enforced in code — not just as an LLM prompt instruction that can be ignored.

**Depends on:** [derive-pipeline-is-defensive](derive.md#derive-pipeline-is-defensive), [derive-pipeline-is-production-ready](derive.md#derive-pipeline-is-production-ready)

### derive-pipeline-is-defensive
**Status:** IN

The derive pipeline applies multiple defensive measures: fail-soft validation, Jaccard-based retraction guard, and environment variable stripping to prevent recursive spawning

**Depends on:** [derive-fail-soft-validation](derive.md#derive-fail-soft-validation), [derive-retraction-guard-uses-jaccard](derive.md#derive-retraction-guard-uses-jaccard), [derive-strips-claudecode-env](derive.md#derive-strips-claudecode-env)
**Supports:** [all-llm-interactions-are-bounded-and-fail-soft](llm.md#all-llm-interactions-are-bounded-and-fail-soft), [derive-pipeline-achieves-end-to-end-fault-tolerance](derive.md#derive-pipeline-achieves-end-to-end-fault-tolerance), [derive-pipeline-has-end-to-end-quality-enforcement](derive.md#derive-pipeline-has-end-to-end-quality-enforcement), [derive-pipeline-is-production-ready](derive.md#derive-pipeline-is-production-ready), [derive-pipeline-is-safe-and-complete](derive.md#derive-pipeline-is-safe-and-complete), [derive-quality-is-comprehensively-code-enforced](derive.md#derive-quality-is-comprehensively-code-enforced), [external-belief-ingestion-is-defensively-layered](external.md#external-belief-ingestion-is-defensively-layered), [llm-driven-mutations-are-safely-bounded](llm.md#llm-driven-mutations-are-safely-bounded)

### derive-pipeline-is-error-accumulating
**Status:** IN

Covered by existing `derive-fail-soft-validation` (validation returns skipped entries with reasons) and `derive-pipeline-is-defensive` (broader defensive characterization)


### derive-pipeline-is-exhaustive-and-terminating
**Status:** OUT

The derive pipeline supports exhaustive exploration mode while guaranteeing termination: `--exhaust` enables automatic application of all discovered proposals, and depth-based cycle detection with pre-recursion memoization prevents infinite loops even when justification chains are cyclic.

**Depends on:** [derive-depth-cycle-guard](derive.md#derive-depth-cycle-guard), [exhaust-implies-auto](other.md#exhaust-implies-auto)
**Supports:** [derive-pipeline-is-safe-and-complete](derive.md#derive-pipeline-is-safe-and-complete), [reasoning-is-exhaustively-deterministic](deterministic.md#reasoning-is-exhaustively-deterministic), [self-correction-is-exhaustive-across-lifecycle](self.md#self-correction-is-exhaustive-across-lifecycle)

### derive-pipeline-is-production-ready
**Status:** OUT

The derive pipeline correctly allocates budgets, validates proposals defensively, and produces well-formed beliefs through a round-trippable prompt contract.

**Depends on:** [derive-pipeline-is-defensive](derive.md#derive-pipeline-is-defensive), [derive-prompt-roundtrips-through-parser](derive.md#derive-prompt-roundtrips-through-parser)
**Supports:** [belief-currency-is-actively-managed](other.md#belief-currency-is-actively-managed), [derive-pipeline-has-complete-coverage](derive.md#derive-pipeline-has-complete-coverage), [derive-pipeline-has-end-to-end-quality-enforcement](derive.md#derive-pipeline-has-end-to-end-quality-enforcement)

### derive-pipeline-is-reproducible-and-defense-in-depth
**Status:** IN

The derive pipeline achieves both reproducibility (deterministic sampling with fixed seeds and accurate budget allocation) and defense-in-depth at the application stage (validation-before-apply trust boundary with per-proposal error isolation), ensuring pipeline runs are repeatable and any surviving bad proposal cannot corrupt sibling proposals.

**Depends on:** [derive-apply-is-isolated-and-caller-validated](derive.md#derive-apply-is-isolated-and-caller-validated), [derive-prompt-is-deterministic-and-reproducible](derive.md#derive-prompt-is-deterministic-and-reproducible)
**Supports:** [derive-pipeline-is-reproducible-and-fully-assured](derive.md#derive-pipeline-is-reproducible-and-fully-assured)

### derive-pipeline-is-reproducible-and-fully-assured
**Status:** OUT

The derive pipeline achieves quadruple assurance: reproducibility (deterministic sampling with fixed seeds and accurate budget allocation), safety (fail-soft validation, Jaccard retraction guards, environment isolation), completeness (exhaustive exploration with guaranteed termination), and efficiency (O(N) budget accumulation with guaranteed floor) — four independently established properties reinforcing pipeline trustworthiness.

**Depends on:** [derive-pipeline-is-reproducible-and-defense-in-depth](derive.md#derive-pipeline-is-reproducible-and-defense-in-depth), [derive-pipeline-is-safe-complete-and-efficient](derive.md#derive-pipeline-is-safe-complete-and-efficient)
**Supports:** [deterministic-reasoning-is-boundary-safe-and-reproducible](deterministic.md#deterministic-reasoning-is-boundary-safe-and-reproducible)

### derive-pipeline-is-safe-and-complete
**Status:** OUT

The derive pipeline simultaneously provides safety (fail-soft validation, Jaccard retraction guards, environment isolation) and completeness (exhaustive exploration with guaranteed cycle-free termination), ensuring LLM-driven belief generation discovers all derivable conclusions without corruption risk.

**Depends on:** [derive-pipeline-is-defensive](derive.md#derive-pipeline-is-defensive), [derive-pipeline-is-exhaustive-and-terminating](derive.md#derive-pipeline-is-exhaustive-and-terminating)
**Supports:** [derive-pipeline-has-complete-coverage](derive.md#derive-pipeline-has-complete-coverage), [derive-pipeline-is-safe-complete-and-efficient](derive.md#derive-pipeline-is-safe-complete-and-efficient), [derived-belief-pipeline-achieves-code-enforced-quality](other.md#derived-belief-pipeline-achieves-code-enforced-quality)

### derive-pipeline-is-safe-complete-and-efficient
**Status:** OUT

The derive pipeline simultaneously achieves safety (fail-soft validation with Jaccard retraction guards and environment isolation), completeness (exhaustive exploration with guaranteed termination via cycle guards), and efficiency (linear O(N) budget accumulation with a floor of 5 beliefs per agent preventing representation starvation).

**Depends on:** [derive-budget-is-efficient-and-floor-bounded](derive.md#derive-budget-is-efficient-and-floor-bounded), [derive-pipeline-is-safe-and-complete](derive.md#derive-pipeline-is-safe-and-complete)
**Supports:** [derive-pipeline-is-reproducible-and-fully-assured](derive.md#derive-pipeline-is-reproducible-and-fully-assured), [llm-belief-operations-span-creation-and-classification](llm.md#llm-belief-operations-span-creation-and-classification), [resource-efficiency-spans-full-pipeline](spans.md#resource-efficiency-spans-full-pipeline)

### derive-prompt-is-deterministic-and-reproducible
**Status:** IN

The derive pipeline's prompt construction is fully reproducible: deterministic sampling with fixed seeds selects consistent belief subsets, and accurate proportional budget allocation ensures each agent receives the same token share across runs.

**Depends on:** [derive-budget-allocation-is-accurate](derive.md#derive-budget-allocation-is-accurate), [sample-mode-is-deterministic](deterministic.md#sample-mode-is-deterministic)
**Supports:** [cluster-derive-is-semantically-informed-and-deterministic](derive.md#cluster-derive-is-semantically-informed-and-deterministic), [derive-achieves-flexibility-with-reproducibility](derive.md#derive-achieves-flexibility-with-reproducibility), [derive-pipeline-achieves-end-to-end-fault-tolerance](derive.md#derive-pipeline-achieves-end-to-end-fault-tolerance), [derive-pipeline-is-reproducible-and-defense-in-depth](derive.md#derive-pipeline-is-reproducible-and-defense-in-depth)

### derive-prompt-round-trippable
**Status:** IN

`write_proposals_file` output can be parsed back by `parse_proposals` with no data loss, enabling a write-review-accept cycle where humans edit proposals in the same format the parser reads


### derive-prompt-roundtrips-through-parser
**Status:** IN

The `### DERIVE` / `### GATE` format is a shared contract between `DERIVE_PROMPT` LLM output, `parse_proposals()` input, and `write_proposals_file()` output, forming a closed serialization loop

**Supports:** [derive-budget-allocation-is-accurate](derive.md#derive-budget-allocation-is-accurate), [derive-pipeline-is-production-ready](derive.md#derive-pipeline-is-production-ready), [derive-quality-is-comprehensively-code-enforced](derive.md#derive-quality-is-comprehensively-code-enforced)

### derive-quality-is-comprehensively-code-enforced
**Status:** OUT

All derive pipeline quality constraints — structural validation, retraction guards, environment isolation, format contracts, AND minimum-antecedent requirements — are enforced through code-level validation, ensuring no invalid proposals can reach the database regardless of LLM prompt compliance.

**Depends on:** [derive-pipeline-is-defensive](derive.md#derive-pipeline-is-defensive), [derive-prompt-roundtrips-through-parser](derive.md#derive-prompt-roundtrips-through-parser)

### derive-report-json-has-rounds-array
**Status:** IN

The `reasons derive --auto --report-dir` command writes a JSON report containing a `rounds` array where each entry has `proposals_found` and `added` counts.


### derive-reports-survive-partial-runs
**Status:** IN

`cmd_derive` and `cmd_review_beliefs` write partial JSON reports after each round/batch via `_write_derive_report`, so crash recovery is possible from the last completed step.

**Supports:** [derive-resilience-preserves-progress-on-error](derive.md#derive-resilience-preserves-progress-on-error)

### derive-resilience-preserves-progress-on-error
**Status:** IN

The derive pipeline is resilient to partial failures: partial results are persisted via JSON reports after each round, and error states are signaled through return codes (-1 for error, 0 for saturation, positive for progress) rather than exceptions, enabling callers to recover and resume.

**Depends on:** [derive-reports-survive-partial-runs](derive.md#derive-reports-survive-partial-runs), [derive-returns-negative-one-on-error](derive.md#derive-returns-negative-one-on-error)
**Supports:** [derive-pipeline-achieves-end-to-end-fault-tolerance](derive.md#derive-pipeline-achieves-end-to-end-fault-tolerance)

### derive-retraction-guard-uses-jaccard
**Status:** IN

`validate_proposals` rejects any proposed belief ID with Jaccard similarity >= 0.5 to an existing OUT node, preventing re-derivation of retracted beliefs

**Supports:** [derive-pipeline-is-defensive](derive.md#derive-pipeline-is-defensive)

### derive-returns-negative-one-on-error
**Status:** IN

`_derive_one_round` returns -1 on error, 0 on saturation, and a positive int for the number of beliefs added, which `cmd_derive` uses to control the exhaust loop.

**Supports:** [derive-resilience-preserves-progress-on-error](derive.md#derive-resilience-preserves-progress-on-error)

### derive-round-returns-negative-on-error
**Status:** IN

`_derive_one_round` returns -1 on error, 0 on saturation, and a positive count on success — using return values instead of exceptions for flow control in the exhaust loop.


### derive-strips-claudecode-env
**Status:** IN

`_derive_one_round` explicitly removes the `CLAUDECODE` environment variable before spawning the model subprocess, preventing recursive Claude Code invocation.

**Supports:** [derive-pipeline-is-defensive](derive.md#derive-pipeline-is-defensive), [llm-integration-is-bounded-fail-soft-and-process-isolated](llm.md#llm-integration-is-bounded-fail-soft-and-process-isolated)

### derive-unused-imports
**Status:** IN

Dead code observation (`subprocess`, `sys`, `Path` imported but unused) — unstable detail that will change when cleaned up.


### derive-uses-subprocess-not-sdk
**Status:** IN

The derive command invokes LLMs by shelling out to `claude` or `gemini` CLI binaries via `asyncio.create_subprocess_exec`, not through any Python SDK.

**Supports:** [all-external-execution-is-subprocess-isolated](external.md#all-external-execution-is-subprocess-isolated)

### derive-validate-before-apply
**Status:** IN

`apply_proposals` trusts its input unconditionally; callers must run `validate_proposals` first or risk database errors from missing antecedents or duplicate IDs.

**Supports:** [derive-apply-is-isolated-and-caller-validated](derive.md#derive-apply-is-isolated-and-caller-validated)

### derive-validate-blocks-retracted-rediscovery
**Status:** IN

`validate_proposals` rejects any proposal whose ID has >= 50% Jaccard token overlap (tokenized on hyphens/colons) with an existing OUT belief, preventing re-derivation of previously retracted conclusions


### derive-validate-rejects-duplicate-ids
**Status:** IN

`validate_proposals` rejects any proposal whose belief ID already exists in the network, preventing overwrites of existing beliefs through the derive pipeline.


### derive-validate-rejects-similar-to-out
**Status:** IN

Covered by existing `derive-retraction-guard-uses-jaccard` which captures the same invariant

