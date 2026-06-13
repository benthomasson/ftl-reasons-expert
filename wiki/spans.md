# spans

[Back to index](index.md)

### belief-consistency-spans-structural-and-semantic-dimensions
**Status:** IN

The system maintains belief consistency through two complementary mechanisms targeting different quality dimensions: deduplication resolves structural redundancy (merging equivalent beliefs while preserving topology with user-editable auditable plans), while contradiction management resolves semantic inconsistency (traceable dependency-directed backtracking with consistent nogood IDs and minimal-disruption culprit selection).

**Depends on:** [contradiction-management-is-complete-and-traceable](complete.md#contradiction-management-is-complete-and-traceable), [dedup-is-topology-preserving-and-auditable](topology.md#dedup-is-topology-preserving-and-auditable)

### data-integrity-spans-architecture
**Status:** OUT

Data integrity is enforced across all three architectural layers: clean layer boundaries prevent cross-cutting mutations, snapshot persistence ensures atomic state transitions, and conservative staleness checking gates CI pipelines.

**Depends on:** [persistence-is-snapshot-not-incremental](other.md#persistence-is-snapshot-not-incremental), [staleness-is-conservative-ci-gate](other.md#staleness-is-conservative-ci-gate), [three-layer-stack-has-clean-boundaries](other.md#three-layer-stack-has-clean-boundaries)
**Supports:** [integrity-enforced-across-architecture-and-lifecycle](lifecycle.md#integrity-enforced-across-architecture-and-lifecycle), [multi-agent-safety-spans-all-layers](agent.md#multi-agent-safety-spans-all-layers), [system-achieves-full-correctness](system.md#system-achieves-full-correctness)

### defense-in-depth-spans-llm-and-system-boundaries
**Status:** OUT

Defense-in-depth is enforced at every external interface through two independently-established layers: LLM integration applies layered defenses across application and process isolation boundaries (bounded execution, fail-soft handling, subprocess isolation), while all system boundaries simultaneously enforce strict validation, evolution tolerance, and resource constraints.

**Depends on:** [llm-integration-is-defense-in-depth-across-layers](llm.md#llm-integration-is-defense-in-depth-across-layers), [system-boundary-enforcement-spans-validation-resilience-and-resources](system.md#system-boundary-enforcement-spans-validation-resilience-and-resources)
**Supports:** [defense-in-depth-is-resource-efficient](other.md#defense-in-depth-is-resource-efficient), [operational-safety-is-defense-in-depth-reinforced](other.md#operational-safety-is-defense-in-depth-reinforced)

### determinism-spans-revision-semantics-through-source-integrity
**Status:** IN

Determinism is maintained end-to-end from belief revision semantics through source integrity verification: revision operations produce deterministic traceable state transitions (structured before/after diffs with controlled irreversibility), AND the source verification that triggers revision actions is itself deterministic and architecturally grounded (fail-safe path resolution, exact SHA-256 comparison, clean layer boundaries) — no step in the revision-triggered-by-source-change pathway introduces non-determinism.

**Depends on:** [revision-semantics-are-deterministic-and-traceable](revision.md#revision-semantics-are-deterministic-and-traceable), [source-integrity-is-deterministic-and-architecturally-grounded](source.md#source-integrity-is-deterministic-and-architecturally-grounded)
**Supports:** [rich-governance-determinism-spans-revision-through-source](governance.md#rich-governance-determinism-spans-revision-through-source)

### dispute-resolution-spans-all-origins
**Status:** OUT

Both dispute resolution mechanisms (intentional dialectical and automated contradiction) are complete and reliable, and all belief origins participate in the same deterministic revision pipeline — every belief from any source can be disputed and resolved through identical mechanisms.

**Depends on:** [all-belief-origins-share-deterministic-revision](deterministic.md#all-belief-origins-share-deterministic-revision), [dispute-resolution-is-complete-and-reliable](complete.md#dispute-resolution-is-complete-and-reliable)
**Supports:** [corrections-span-all-origins-with-full-auditability](other.md#corrections-span-all-origins-with-full-auditability)

### dual-quality-enforcement-spans-automated-and-explicit
**Status:** OUT

Belief quality is enforced by two independent mechanisms that cannot interfere: automated self-correction autonomously maintains consistency through contradiction resolution and staleness detection, while explicit quality review independently evaluates derived beliefs with read-only fault tolerance — dual enforcement ensures quality even when one mechanism is insufficient.

**Depends on:** [complete-system-is-self-correcting](complete.md#complete-system-is-self-correcting), [review-is-read-only-and-fault-tolerant](other.md#review-is-read-only-and-fault-tolerant)
**Supports:** [complete-quality-lifecycle-spans-creation-through-egress](complete.md#complete-quality-lifecycle-spans-creation-through-egress)

### edge-case-safety-spans-creation-and-maintenance
**Status:** OUT

The system handles edge cases safely across both temporal dimensions: at creation time, uniform revision covers all semantic edge cases (vacuous premises, asymmetric absence, empty antecedents) through minimal primitives; at maintenance time, contradiction resolution and staleness detection actively catch drift — no edge case is safe only at one point in time.

**Depends on:** [edge-case-uniformity-follows-from-minimality](other.md#edge-case-uniformity-follows-from-minimality), [self-correction-spans-creation-and-maintenance](self.md#self-correction-spans-creation-and-maintenance)
**Supports:** [invariants-hold-across-origin-and-time](other.md#invariants-hold-across-origin-and-time), [revision-coverage-requires-sound-propagation](revision.md#revision-coverage-requires-sound-propagation)

### exception-safety-spans-tms-and-source-lifecycle
**Status:** IN

The system handles failures safely across both domains: the TMS core resolves exceptional conditions (contradictions via backtracking, challenges via deterministic propagation) while the source integrity pipeline handles its failure modes (missing files, changed content, deleted sources) — no exception in either domain can produce inconsistent or undefined state.

**Depends on:** [all-exceptions-are-safely-handled](other.md#all-exceptions-are-safely-handled), [source-lifecycle-is-fail-safe-and-gapless](source.md#source-lifecycle-is-fail-safe-and-gapless)
**Supports:** [rich-governance-is-exception-safe](governance.md#rich-governance-is-exception-safe), [source-to-tms-integrity-is-deterministic-and-exception-safe](source.md#source-to-tms-integrity-is-deterministic-and-exception-safe)

### fault-tolerance-spans-inspection-through-self-correction
**Status:** OUT

Fault tolerance covers the complete belief quality spectrum: passive inspection operations (review, staleness checking, list-negative classification) degrade gracefully with fail-safe defaults and never mutate state, AND active self-correction (contradiction resolution via backtracking, staleness detection via source hashing) continues operating without external LLM dependencies — the system maintains quality assurance autonomously even when LLMs, external files, or network resources are unavailable.

**Depends on:** [all-belief-inspection-is-non-mutating-and-fault-tolerant](other.md#all-belief-inspection-is-non-mutating-and-fault-tolerant), [self-correction-is-resilient-to-llm-unavailability](self.md#self-correction-is-resilient-to-llm-unavailability)
**Supports:** [quality-lifecycle-is-fault-tolerant-and-resource-efficient](lifecycle.md#quality-lifecycle-is-fault-tolerant-and-resource-efficient)

### format-resilience-spans-all-external-interfaces
**Status:** OUT

The system tolerates format variation at every external interface — derive output parsers support version fallback, import parsers silently skip unknown fields, storage handles schema evolution — and extends this resilience to LLM response parsing, where the list-negative parser uses regex extraction to recover structured data from prose-laden responses.

**Depends on:** [list-negative-parser-is-fully-resilient](other.md#list-negative-parser-is-fully-resilient), [system-tolerates-evolution-at-all-boundaries](system.md#system-tolerates-evolution-at-all-boundaries)
**Supports:** [external-ingestion-is-format-resilient-and-defensively-layered](external.md#external-ingestion-is-format-resilient-and-defensively-layered), [format-resilient-boundaries-enforce-validated-trust](other.md#format-resilient-boundaries-enforce-validated-trust)

### minimality-spans-computation-and-revision
**Status:** OUT

Minimality is the shared generative root of both forward and backward system properties: forward computation achieves uniformity and determinism, backward revision achieves universal safety covering all edge cases and lifecycle states — the same minimal primitives produce correctness in both directions without requiring separate design efforts or independent correctness arguments.

**Depends on:** [minimality-produces-uniformity-and-determinism](other.md#minimality-produces-uniformity-and-determinism), [revision-is-universally-safe](revision.md#revision-is-universally-safe)
**Supports:** [minimality-sustains-closed-loop-maintenance](other.md#minimality-sustains-closed-loop-maintenance)

### mutation-safety-spans-all-dimensions
**Status:** OUT

Every mutation is safe along three orthogonal dimensions simultaneously: source dimension (human, LLM, or agent), semantic dimension (uniform revision via outlist defeat and backtracking), and lifecycle dimension (respects retraction state and propagation bounds) — no mutation path is exempt from any dimension.

**Depends on:** [all-mutation-sources-are-safe-and-uniform](safe.md#all-mutation-sources-are-safe-and-uniform), [revision-spans-lifecycle-and-all-sources](revision.md#revision-spans-lifecycle-and-all-sources)

### operational-profile-spans-all-backends
**Status:** OUT

The safe, assured, resource-bounded operational profile holds identically across all storage backends — the same safety, assurance, and resource-efficiency guarantees apply uniformly to both SQLite and PostgreSQL deployments.

**Depends on:** [operational-profile-is-safe-assured-and-resource-bounded](safe.md#operational-profile-is-safe-assured-and-resource-bounded), [safety-is-enforced-across-all-layers-and-backends](other.md#safety-is-enforced-across-all-layers-and-backends)

### operational-safety-spans-all-mutation-sources
**Status:** OUT

Operational safety extends across both internal mutation pipelines (atomic load/save transactions, deterministic propagation, write-flag gating) and external belief ingestion pathways (defensive validation, Jaccard retraction guards, environment isolation, agent namespace containment), ensuring every mutation source — whether programmatic, LLM-driven, or agent-imported — passes through integrity enforcement.

**Depends on:** [all-external-belief-paths-are-safely-bounded](external.md#all-external-belief-paths-are-safely-bounded), [operational-integrity-is-end-to-end](other.md#operational-integrity-is-end-to-end)
**Supports:** [integrity-is-boundary-and-source-agnostic](source.md#integrity-is-boundary-and-source-agnostic)

### resource-efficiency-spans-footprint-through-budgets
**Status:** OUT

The system achieves resource efficiency from the broadest to the narrowest scope: zero external dependencies and lazy module loading minimize the static footprint at packaging and startup, while efficient O(1) budget tracking with approximate token estimation constrains resource consumption during both compact distillation and derive belief allocation at runtime.

**Depends on:** [budget-enforcement-is-efficient-across-pipeline](other.md#budget-enforcement-is-efficient-across-pipeline), [system-resource-footprint-is-minimal-at-all-phases](system.md#system-resource-footprint-is-minimal-at-all-phases)

### resource-efficiency-spans-full-pipeline
**Status:** OUT

Resource efficiency is enforced across the complete operational pipeline: from packaging and startup (zero external dependencies with lazy loading) through belief derivation (linear O(N) budget allocation with floor bounds) to output generation (O(1) per-line budget tracking with bounded pure compact summaries), ensuring minimal resource consumption at every phase

**Depends on:** [derive-pipeline-is-safe-complete-and-efficient](derive.md#derive-pipeline-is-safe-complete-and-efficient), [system-efficiency-spans-packaging-and-runtime](system.md#system-efficiency-spans-packaging-and-runtime)
**Supports:** [defense-in-depth-is-resource-efficient](other.md#defense-in-depth-is-resource-efficient), [operational-assurance-is-resource-efficient](other.md#operational-assurance-is-resource-efficient), [quality-lifecycle-is-complete-and-resource-efficient](lifecycle.md#quality-lifecycle-is-complete-and-resource-efficient), [self-correction-operates-within-efficient-pipeline](self.md#self-correction-operates-within-efficient-pipeline)

### verified-production-correctness-spans-all-origins
**Status:** OUT

Verified production correctness extends universally across all belief origins: the complete architecture achieves verified correctness with deterministic state trajectories and lifecycle-complete monitoring, AND every state change for every belief — including externally-originated ones at full integration parity — follows a deterministic path with complete traceability.

**Depends on:** [complete-architecture-achieves-verified-production-correctness](complete.md#complete-architecture-achieves-verified-production-correctness), [deterministic-history-extends-to-all-origins](deterministic.md#deterministic-history-extends-to-all-origins)
**Supports:** [verified-correctness-is-independently-observable](other.md#verified-correctness-is-independently-observable), [verified-correctness-is-permanently-documented](other.md#verified-correctness-is-permanently-documented)
