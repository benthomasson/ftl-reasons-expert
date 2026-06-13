# system

[Back to index](index.md)

### full-system-integrity-is-gap-free
**Status:** OUT

The system achieves gap-free integrity — enforced across all architectural layers, lifecycle states, and mutation paths — only when the dependents reverse index is reliably maintained and propagation handles dangling references gracefully.

**Depends on:** [integrity-enforced-across-architecture-and-lifecycle](lifecycle.md#integrity-enforced-across-architecture-and-lifecycle), [mutations-are-atomic-and-safely-propagated](other.md#mutations-are-atomic-and-safely-propagated)

### non-monotonic-system-is-single-reversible-primitive
**Status:** IN

The entire non-monotonic reasoning system — challenges, kill-switches, supersession, and dialectics — is built on a single primitive (outlist) that is inherently reversible, with no dedicated machinery for any defeat pattern.

**Depends on:** [all-defeat-mechanisms-are-reversible](other.md#all-defeat-mechanisms-are-reversible), [dialectical-structure-is-recursive-outlist](other.md#dialectical-structure-is-recursive-outlist), [outlist-is-universal-defeat-mechanism](other.md#outlist-is-universal-defeat-mechanism)
**Supports:** [belief-revision-is-comprehensive-and-minimal](revision.md#belief-revision-is-comprehensive-and-minimal), [belief-revision-is-fully-reliable](revision.md#belief-revision-is-fully-reliable), [reasoning-engine-is-deterministic-and-reversible](deterministic.md#reasoning-engine-is-deterministic-and-reversible), [system-achieves-full-correctness](system.md#system-achieves-full-correctness), [system-semantics-are-minimal-and-complete](system.md#system-semantics-are-minimal-and-complete)

### system-achieves-full-correctness
**Status:** OUT

The system achieves correctness at every level: deterministic conservative truth maintenance, a single reversible primitive for all non-monotonic features, and data integrity spanning all architectural layers — the system is sound end-to-end.

**Depends on:** [data-integrity-spans-architecture](spans.md#data-integrity-spans-architecture), [non-monotonic-system-is-single-reversible-primitive](system.md#non-monotonic-system-is-single-reversible-primitive), [tms-core-is-deterministic-and-conservative](deterministic.md#tms-core-is-deterministic-and-conservative)

### system-achieves-tripartite-operational-assurance
**Status:** OUT

The system achieves a complete operational profile across three independent assurance dimensions: externally controlled (bounded interfaces with defensive ingestion), internally self-correcting (contradiction resolution and staleness detection), and query-resilient (graceful degradation across all access paths) — ensuring no operational scenario is unaddressed.

**Depends on:** [query-resilience-serves-self-correcting-knowledge](self.md#query-resilience-serves-self-correcting-knowledge), [system-is-externally-controlled-and-internally-self-correcting](system.md#system-is-externally-controlled-and-internally-self-correcting)
**Supports:** [source-grounded-correction-has-tripartite-assurance](source.md#source-grounded-correction-has-tripartite-assurance)

### system-artifacts-maintain-consistent-identification
**Status:** IN

Both automatically-generated dialectical structures (challenge nodes with deterministic auto-ID generation) and contradiction records (nogoods with unconditional recording) maintain consistent, referenceable identification schemes — system-generated artifacts are as addressable as user-created beliefs.

**Depends on:** [challenge-id-auto-generation](other.md#challenge-id-auto-generation), [nogood-resolution-maintains-consistent-ids](other.md#nogood-resolution-maintains-consistent-ids)
**Supports:** [all-identification-is-deterministic-and-collision-free](deterministic.md#all-identification-is-deterministic-and-collision-free), [contradiction-resolution-is-traceable-and-recoverable](other.md#contradiction-resolution-is-traceable-and-recoverable), [mutations-achieve-full-traceability](other.md#mutations-achieve-full-traceability), [self-correction-produces-referenceable-artifacts](self.md#self-correction-produces-referenceable-artifacts), [system-history-is-consistently-referenceable](system.md#system-history-is-consistently-referenceable)

### system-assurance-is-universal-and-multidimensional
**Status:** OUT

The system's operational assurance spans all dimensions — temporal completeness of self-correction, end-to-end reliability of read and write paths, and external control through bidirectional bounds and defensive ingestion — and holds universally across all architectural layers, storage backends, and adverse graph conditions including vacuous premises, asymmetric absence, and empty antecedents.

**Depends on:** [operational-safety-is-universal-and-condition-independent](other.md#operational-safety-is-universal-and-condition-independent), [system-assurance-spans-correction-reliability-and-control](system.md#system-assurance-spans-correction-reliability-and-control)
**Supports:** [growth-preserves-universal-assurance](other.md#growth-preserves-universal-assurance)

### system-assurance-spans-correction-reliability-and-control
**Status:** OUT

The system's operational assurance spans three independent dimensions: temporal completeness of self-correction (creation-time contradiction resolution and maintenance-time staleness detection), resource sustainability (bounded token budgets preventing exhaustion), and external controllability (bidirectional interface bounds with defensive ingestion) — no assurance gap exists along any axis.

**Depends on:** [self-correction-is-temporally-complete-and-resource-sustainable](self.md#self-correction-is-temporally-complete-and-resource-sustainable), [system-is-reliable-self-correcting-and-externally-controlled](system.md#system-is-reliable-self-correcting-and-externally-controlled)
**Supports:** [backend-agnostic-operational-assurance](other.md#backend-agnostic-operational-assurance), [operational-assurance-is-resource-efficient](other.md#operational-assurance-is-resource-efficient), [system-assurance-is-universal-and-multidimensional](system.md#system-assurance-is-universal-and-multidimensional)

### system-autonomously-converges-and-self-corrects
**Status:** OUT

The system autonomously reaches and maintains consistent states through two complementary mechanisms: passive convergence ensures every modification path (import, retraction, dedup) reaches a deterministic stable state, while active self-correction (contradiction resolution and staleness detection) ensures consistency is preserved over time — combining equilibrium-seeking with consistency-maintaining.

**Depends on:** [complete-system-is-self-correcting](complete.md#complete-system-is-self-correcting), [system-reaches-equilibrium-from-all-modification-paths](system.md#system-reaches-equilibrium-from-all-modification-paths)
**Supports:** [autonomous-convergence-preserves-trust-boundaries](other.md#autonomous-convergence-preserves-trust-boundaries), [convergence-produces-evaluation-invariant-equilibria](other.md#convergence-produces-evaluation-invariant-equilibria)

### system-boundaries-are-evolution-tolerant-and-reference-safe
**Status:** OUT

The system handles boundary interactions safely along two independent dimensions: format and schema evolution is tolerated gracefully (derive parser fallbacks, forward-compatible metadata lines, SQLite schema migration via try/except), while reference validation prevents invalid IDs from crossing any boundary (import normalization drops unknown refs, nogoods skip missing nodes, LLM hallucination filtering discards phantom IDs)

**Depends on:** [reference-validation-is-defense-in-depth](other.md#reference-validation-is-defense-in-depth), [system-tolerates-evolution-at-all-boundaries](system.md#system-tolerates-evolution-at-all-boundaries)
**Supports:** [references-are-durable-across-persistence-and-evolution](other.md#references-are-durable-across-persistence-and-evolution)

### system-boundaries-are-validating-and-evolution-tolerant
**Status:** OUT

All system boundaries simultaneously enforce strict input validation (typed exceptions, referential integrity checks, hallucination filtering) and tolerate evolution gracefully (dual format parsers, forward-compatible metadata, schema-tolerant loading) — boundaries are both strict about current invariants and adaptive to future changes.

**Depends on:** [input-validation-is-comprehensive-at-all-boundaries](other.md#input-validation-is-comprehensive-at-all-boundaries), [system-tolerates-evolution-at-all-boundaries](system.md#system-tolerates-evolution-at-all-boundaries)
**Supports:** [deterministic-reasoning-within-evolution-tolerant-boundaries](deterministic.md#deterministic-reasoning-within-evolution-tolerant-boundaries), [format-resilient-boundaries-enforce-validated-trust](other.md#format-resilient-boundaries-enforce-validated-trust), [system-boundary-enforcement-spans-validation-resilience-and-resources](system.md#system-boundary-enforcement-spans-validation-resilience-and-resources)

### system-boundary-enforcement-spans-validation-resilience-and-resources
**Status:** OUT

All system boundaries simultaneously enforce three independent properties: strict input validation through typed exceptions and referential integrity checks that reject malformed or dangling references, forward-compatible resilience that tolerates format and schema evolution without requiring coordinated upgrades, and resource governance through accurate bidirectional token budgets with transitive subset-gated access control — boundaries serve as gates for correctness, adapters for evolution, and constraints on resource consumption.

**Depends on:** [information-pipeline-is-resource-governed-and-access-controlled](other.md#information-pipeline-is-resource-governed-and-access-controlled), [system-boundaries-are-validating-and-evolution-tolerant](system.md#system-boundaries-are-validating-and-evolution-tolerant)
**Supports:** [defense-in-depth-spans-llm-and-system-boundaries](spans.md#defense-in-depth-spans-llm-and-system-boundaries)

### system-converges-from-addition-and-removal
**Status:** OUT

The system reaches deterministic stable states from both directions: import reconciliation converges through fixpoint iteration and dual reconciliation modes when beliefs are added, while retraction cascades terminate through BFS with stop-on-unchanged when beliefs are removed — bidirectional convergence guarantees that no sequence of additions or removals leaves the network in an oscillating or indeterminate state.

**Depends on:** [import-reconciliation-converges-deterministically](import.md#import-reconciliation-converges-deterministically), [retraction-cascade-is-transitive-and-terminating](other.md#retraction-cascade-is-transitive-and-terminating)
**Supports:** [system-reaches-equilibrium-from-all-modification-paths](system.md#system-reaches-equilibrium-from-all-modification-paths)

### system-efficiency-spans-packaging-and-runtime
**Status:** OUT

Resource efficiency is enforced at every system phase: zero external dependencies with lazy loading minimize the static footprint at packaging and startup, while O(1) per-line budget tracking with chars/4 token estimation minimize computational overhead during runtime belief distillation.

**Depends on:** [compact-is-efficient-deterministic-and-bounded](compact.md#compact-is-efficient-deterministic-and-bounded), [system-resource-footprint-is-minimal-at-all-phases](system.md#system-resource-footprint-is-minimal-at-all-phases)
**Supports:** [resource-efficiency-spans-full-pipeline](spans.md#resource-efficiency-spans-full-pipeline)

### system-guarantees-are-universal-and-permanent
**Status:** OUT

The system's ultimate properties are both permanent (maintained indefinitely without temporal degradation) and universal (extending fully to all beliefs regardless of origin) — no belief can escape the system's guarantees along either the time or scope dimension.

**Depends on:** [system-properties-are-indefinitely-maintained](system.md#system-properties-are-indefinitely-maintained), [system-properties-extend-fully-to-external-beliefs](system.md#system-properties-extend-fully-to-external-beliefs)
**Supports:** [resource-efficient-guarantees-are-universal-and-permanent](other.md#resource-efficient-guarantees-are-universal-and-permanent), [system-guarantees-are-universal-permanent-and-verifiable](system.md#system-guarantees-are-universal-permanent-and-verifiable)

### system-guarantees-are-universal-permanent-and-verifiable
**Status:** OUT

The system's ultimate guarantees are simultaneously universal (extending fully to all belief types including externally-integrated ones), permanent (maintained indefinitely without temporal degradation), and independently verifiable (origin-agnostic observability enables external audit without requiring trust in the system's internal state).

**Depends on:** [origin-agnostic-guarantees-are-verifiable-and-self-sustaining](self.md#origin-agnostic-guarantees-are-verifiable-and-self-sustaining), [system-guarantees-are-universal-and-permanent](system.md#system-guarantees-are-universal-and-permanent)

### system-history-is-consistently-referenceable
**Status:** OUT

Every event in the system's operational history follows a deterministic path with complete traceability AND produces consistently-identifiable artifacts (auto-generated challenge IDs, unconditionally-recorded nogoods), making the complete history both causally traceable and individually referenceable by stable identifiers.

**Depends on:** [complete-system-history-is-deterministic-and-traceable](complete.md#complete-system-history-is-deterministic-and-traceable), [system-artifacts-maintain-consistent-identification](system.md#system-artifacts-maintain-consistent-identification)
**Supports:** [autonomous-convergence-produces-documented-equilibria](other.md#autonomous-convergence-produces-documented-equilibria), [equilibrium-trajectory-is-deterministic-and-referenceable](deterministic.md#equilibrium-trajectory-is-deterministic-and-referenceable), [self-correction-is-exhaustive-and-artifact-producing](self.md#self-correction-is-exhaustive-and-artifact-producing)

### system-is-externally-controlled-and-internally-self-correcting
**Status:** OUT

The system achieves dual-layer assurance: external interfaces are fully controlled through bidirectional token bounds and defensive belief ingestion, while internal consistency is actively maintained through contradiction resolution at derivation time and staleness detection at maintenance time.

**Depends on:** [complete-system-is-self-correcting](complete.md#complete-system-is-self-correcting), [external-surface-is-fully-controlled](external.md#external-surface-is-fully-controlled)
**Supports:** [system-achieves-tripartite-operational-assurance](system.md#system-achieves-tripartite-operational-assurance), [system-is-reliable-self-correcting-and-externally-controlled](system.md#system-is-reliable-self-correcting-and-externally-controlled)

### system-is-fully-characterized-self-maintaining-loop
**Status:** OUT

The closed maintenance loop is fully characterized along three independent dimensions: it operates identically regardless of belief origin, every self-correction leaves traceable history, and minimality generates the mechanisms that sustain the loop itself — no dimension of the loop's behavior is unspecified or opaque.

**Depends on:** [closed-loop-is-origin-agnostic](other.md#closed-loop-is-origin-agnostic), [maintenance-loop-is-fully-observable](other.md#maintenance-loop-is-fully-observable), [minimality-is-self-sustaining](self.md#minimality-is-self-sustaining)
**Supports:** [fully-characterized-loop-sustains-indefinitely](other.md#fully-characterized-loop-sustains-indefinitely), [self-maintenance-is-fully-auditable](self.md#self-maintenance-is-fully-auditable), [system-is-self-sustaining-and-invariant-preserving](system.md#system-is-self-sustaining-and-invariant-preserving)

### system-is-minimal-sound-and-scalable
**Status:** OUT

The entire system — from single-node truth semantics through multi-agent operation — achieves semantic minimality (all features derive from uniform primitives), operational soundness (deterministic reversible truth maintenance), and safe scalability (isolated multi-agent operation) simultaneously.

**Depends on:** [multi-agent-reasoning-is-sound-and-scalable](agent.md#multi-agent-reasoning-is-sound-and-scalable), [system-semantics-are-minimal-and-complete](system.md#system-semantics-are-minimal-and-complete)
**Supports:** [integrity-and-scalability-are-complementary](other.md#integrity-and-scalability-are-complementary), [system-is-unified-minimal-dialectical-and-scalable](system.md#system-is-unified-minimal-dialectical-and-scalable)

### system-is-reliable-self-correcting-and-externally-controlled
**Status:** OUT

The system achieves triple-layered assurance: reliability spanning internal and external boundaries (both read and write paths reliable, external interface bidirectionally bounded), active self-correction for consistency maintenance (contradiction resolution and staleness detection), and full external surface control through defensive ingestion and token bounds — combining reactive integrity maintenance with proactive boundary enforcement.

**Depends on:** [system-is-externally-controlled-and-internally-self-correcting](system.md#system-is-externally-controlled-and-internally-self-correcting), [system-reliability-spans-internal-and-external](system.md#system-reliability-spans-internal-and-external)
**Supports:** [system-assurance-spans-correction-reliability-and-control](system.md#system-assurance-spans-correction-reliability-and-control)

### system-is-resource-efficient-self-sustaining-and-auditable
**Status:** OUT

The system achieves a self-reinforcing triad: resource efficiency reinforces self-sustainability (preventing exhaustion of the minimality-generated maintenance loop), self-sustainability maintains indefinite auditability (the audit trail never degrades because the maintenance loop that produces it is self-maintaining), and auditability remains resource-efficient (the overhead of permanent traceability does not threaten sustainability) — forming a closed positive-feedback cycle.

**Depends on:** [resource-efficient-self-maintenance-is-indefinitely-auditable](self.md#resource-efficient-self-maintenance-is-indefinitely-auditable), [self-sustainability-is-reinforced-by-resource-efficiency](self.md#self-sustainability-is-reinforced-by-resource-efficiency)
**Supports:** [resource-efficient-guarantees-are-universal-and-permanent](other.md#resource-efficient-guarantees-are-universal-and-permanent)

### system-is-self-correcting-and-exception-proof
**Status:** OUT

The system is both actively self-correcting (maintaining consistency through the derive pipeline for new beliefs and staleness detection for existing ones) and passively exception-proof (handling contradictions through deterministic backtracking and challenges through reliable dialectical transformation) — providing comprehensive fault tolerance that covers both anticipated maintenance and unanticipated disruptions.

**Depends on:** [all-exceptions-are-safely-handled](other.md#all-exceptions-are-safely-handled), [complete-system-is-self-correcting](complete.md#complete-system-is-self-correcting)
**Supports:** [self-correction-is-minimality-enforced](self.md#self-correction-is-minimality-enforced)

### system-is-self-sustaining-and-invariant-preserving
**Status:** OUT

The fully characterized self-maintaining loop not only sustains its own operation through minimality's fixed-point property but also comprehensively preserves all system invariants through both temporal coverage (revision loops) and structural coverage (architectural grounding).

**Depends on:** [invariant-preservation-is-comprehensive](other.md#invariant-preservation-is-comprehensive), [system-is-fully-characterized-self-maintaining-loop](system.md#system-is-fully-characterized-self-maintaining-loop)
**Supports:** [system-is-self-sustaining-auditable-and-invariant-complete](system.md#system-is-self-sustaining-auditable-and-invariant-complete)

### system-is-self-sustaining-auditable-and-invariant-complete
**Status:** OUT

The system simultaneously achieves three ultimate properties: self-sustainability through minimality's fixed-point, comprehensive invariant preservation, and complete operational auditability for every maintenance action — a single closed architecture that maintains itself, preserves all guarantees, and traces every action.

**Depends on:** [self-maintenance-is-fully-auditable](self.md#self-maintenance-is-fully-auditable), [system-is-self-sustaining-and-invariant-preserving](system.md#system-is-self-sustaining-and-invariant-preserving)
**Supports:** [system-properties-are-indefinitely-maintained](system.md#system-properties-are-indefinitely-maintained), [system-properties-extend-fully-to-external-beliefs](system.md#system-properties-extend-fully-to-external-beliefs)

### system-is-unified-minimal-dialectical-and-scalable
**Status:** OUT

The entire system — minimal primitives, sound multi-agent scaling, and complete dialectical revision — forms a unified design where every feature derives from the same core outlist/disjunction semantics with deterministic, reversible behavior.

**Depends on:** [dialectics-complete-the-revision-system](complete.md#dialectics-complete-the-revision-system), [system-is-minimal-sound-and-scalable](system.md#system-is-minimal-sound-and-scalable)
**Supports:** [complete-unified-system-is-production-ready](complete.md#complete-unified-system-is-production-ready), [unified-system-maintains-end-to-end-integrity](system.md#unified-system-maintains-end-to-end-integrity)

### system-operations-never-crash
**Status:** OUT

Every system operation is crash-free: atomic mutations prevent partial state corruption, deterministic reversible reasoning prevents oscillation and ambiguity, and uniform evaluation prevents dispatch errors — provided no dangling dependent references exist in the graph.

**Depends on:** [mutations-are-atomic-and-safely-propagated](other.md#mutations-are-atomic-and-safely-propagated), [reasoning-engine-is-deterministic-and-reversible](deterministic.md#reasoning-engine-is-deterministic-and-reversible)

### system-output-is-complete-bounded-and-ci-ready
**Status:** OUT

The system's two primary output mechanisms — compact belief summaries and staleness reports — both meet production standards: structurally complete with priority ordering, predictably bounded by token budgets, and CI-pipeline ready with deterministic sorted output, nonzero exit codes, and machine-parseable schemas.

**Depends on:** [compact-output-is-structurally-complete-and-predictably-bounded](compact.md#compact-output-is-structurally-complete-and-predictably-bounded), [staleness-output-is-ci-pipeline-ready](other.md#staleness-output-is-ci-pipeline-ready)
**Supports:** [output-governance-is-complete-authorized-and-ci-ready](governance.md#output-governance-is-complete-authorized-and-ci-ready)

### system-output-is-comprehensively-governed
**Status:** OUT

All system output is simultaneously normalized (uniform fail-safe schemas with deterministic structure), authorized (access-tag subset gating with transitive inheritance), and resource-constrained (accurate token budgets enforced bidirectionally) — achieving comprehensive output governance across all independent quality dimensions.

**Depends on:** [all-outputs-are-normalized-deterministic-and-resilient](deterministic.md#all-outputs-are-normalized-deterministic-and-resilient), [information-governance-is-end-to-end-authorized-and-resource-constrained](governance.md#information-governance-is-end-to-end-authorized-and-resource-constrained)
**Supports:** [output-governance-is-comprehensive-and-self-sustaining](governance.md#output-governance-is-comprehensive-and-self-sustaining)

### system-properties-are-indefinitely-maintained
**Status:** OUT

The system's three ultimate properties — self-sustainability, comprehensive auditability, and complete invariant preservation — extend beyond external beliefs to unlimited temporal scope: resource-sustainable self-correction within a deterministic lifecycle ensures these properties hold not just now but indefinitely, provided propagation correctness is maintained.

**Depends on:** [self-correction-sustains-lifecycle-indefinitely](self.md#self-correction-sustains-lifecycle-indefinitely), [system-is-self-sustaining-auditable-and-invariant-complete](system.md#system-is-self-sustaining-auditable-and-invariant-complete)
**Supports:** [system-guarantees-are-universal-and-permanent](system.md#system-guarantees-are-universal-and-permanent)

### system-properties-emerge-from-unified-design
**Status:** OUT

All four primary system properties — integrity, scalability, extensibility, and robustness — emerge from a single unified architectural design rather than requiring independent engineering effort; integrity and scalability are complementary consequences of unified internal/external integrity with sound multi-agent scaling, while extensibility and robustness are jointly yielded by minimality, and the design that produces both pairs is the same.

**Depends on:** [integrity-and-scalability-are-complementary](other.md#integrity-and-scalability-are-complementary), [minimality-yields-extensibility-and-robustness](other.md#minimality-yields-extensibility-and-robustness)
**Supports:** [minimality-is-both-generative-and-unifying](other.md#minimality-is-both-generative-and-unifying)

### system-properties-extend-fully-to-external-beliefs
**Status:** OUT

The system's three ultimate properties — self-sustainability through minimality's fixed-point, comprehensive auditability through fully-characterized maintenance, and complete invariant preservation — extend fully to externally-sourced beliefs through self-sustaining invariant preservation that dynamically encompasses external beliefs as first-class participants.

**Depends on:** [self-sustaining-preservation-encompasses-external-beliefs](self.md#self-sustaining-preservation-encompasses-external-beliefs), [system-is-self-sustaining-auditable-and-invariant-complete](system.md#system-is-self-sustaining-auditable-and-invariant-complete)
**Supports:** [system-guarantees-are-universal-and-permanent](system.md#system-guarantees-are-universal-and-permanent)

### system-reaches-equilibrium-from-all-modification-paths
**Status:** OUT

The system converges to deterministic stable states through every modification path: import achieves ordered convergent reconciliation (add → propagate → retract sequencing with fixpoint convergence), retraction cascades terminate through BFS with stop-on-unchanged, and both addition and removal operations reach equilibrium — no modification can leave the system in a non-convergent state.

**Depends on:** [import-achieves-ordered-convergent-reconciliation](import.md#import-achieves-ordered-convergent-reconciliation), [system-converges-from-addition-and-removal](system.md#system-converges-from-addition-and-removal)
**Supports:** [all-modifications-converge-with-reporting-and-recovery](other.md#all-modifications-converge-with-reporting-and-recovery), [system-autonomously-converges-and-self-corrects](system.md#system-autonomously-converges-and-self-corrects)

### system-reliability-spans-internal-and-external
**Status:** OUT

The system is reliable along both internal and external dimensions: internally, both read paths (comprehensive staleness detection) and write paths (crash-free propagation) are reliable; externally, interfaces are bidirectionally bounded with budget-constrained output and comprehensive staleness-gated input.

**Depends on:** [external-interface-is-bidirectionally-bounded](external.md#external-interface-is-bidirectionally-bounded), [read-and-write-paths-are-both-reliable](other.md#read-and-write-paths-are-both-reliable)
**Supports:** [system-is-reliable-self-correcting-and-externally-controlled](system.md#system-is-reliable-self-correcting-and-externally-controlled)

### system-resource-footprint-is-minimal-at-all-phases
**Status:** OUT

The system achieves minimal resource footprint across all lifecycle phases: zero external dependencies at both packaging and implementation levels eliminate installation overhead and version conflicts, while lazy module imports in both API and CLI layers defer heavy computation until actually needed — minimizing deployment complexity, startup time, and memory consumption simultaneously.

**Depends on:** [project-has-zero-external-coupling](external.md#project-has-zero-external-coupling), [startup-performance-uses-lazy-loading](other.md#startup-performance-uses-lazy-loading)
**Supports:** [resource-efficiency-spans-footprint-through-budgets](spans.md#resource-efficiency-spans-footprint-through-budgets), [self-sustainability-is-reinforced-by-resource-efficiency](self.md#self-sustainability-is-reinforced-by-resource-efficiency), [system-efficiency-spans-packaging-and-runtime](system.md#system-efficiency-spans-packaging-and-runtime)

### system-semantics-are-minimal-and-complete
**Status:** IN

The entire TMS — both monotonic truth maintenance and non-monotonic defeat — derives from a minimal set of uniform primitives: emergent truth rules (disjunction over conjunction, premise-from-absence) combined with a single reversible outlist mechanism that underlies all defeat features, with no additional machinery required.

**Depends on:** [non-monotonic-system-is-single-reversible-primitive](system.md#non-monotonic-system-is-single-reversible-primitive), [truth-semantics-are-emergent-and-uniform](other.md#truth-semantics-are-emergent-and-uniform)
**Supports:** [semantic-minimality-with-operational-determinism](other.md#semantic-minimality-with-operational-determinism), [semantics-and-revision-share-minimal-foundations](revision.md#semantics-and-revision-share-minimal-foundations), [system-is-minimal-sound-and-scalable](system.md#system-is-minimal-sound-and-scalable)

### system-sustainably-grows-and-self-corrects
**Status:** OUT

The system simultaneously grows its knowledge base through exhaustive deterministic reasoning and LLM-driven derivation with guaranteed termination, while sustainably self-correcting through contradiction resolution and staleness detection — all within bounded resource consumption managed by accurate bidirectional token budgets

**Depends on:** [reasoning-and-knowledge-expansion-are-both-exhaustive](other.md#reasoning-and-knowledge-expansion-are-both-exhaustive), [self-correction-is-resource-sustainable](self.md#self-correction-is-resource-sustainable)
**Supports:** [sustainable-growth-is-indefinitely-self-correcting](self.md#sustainable-growth-is-indefinitely-self-correcting)

### system-tolerates-evolution-at-all-boundaries
**Status:** OUT

The system handles format and schema evolution gracefully at every external boundary: derive output parsers support two format versions with automatic fallback, belief import silently skips unknown metadata fields, and storage tolerates missing tables from older database schemas via exception handling

**Depends on:** [derive-parse-supports-two-formats](derive.md#derive-parse-supports-two-formats), [import-beliefs-parser-is-forward-compatible](import.md#import-beliefs-parser-is-forward-compatible), [storage-handles-schema-evolution-via-try-except](other.md#storage-handles-schema-evolution-via-try-except)
**Supports:** [format-resilience-spans-all-external-interfaces](spans.md#format-resilience-spans-all-external-interfaces), [self-correction-is-evolution-tolerant-and-sustainable](self.md#self-correction-is-evolution-tolerant-and-sustainable), [system-boundaries-are-evolution-tolerant-and-reference-safe](system.md#system-boundaries-are-evolution-tolerant-and-reference-safe), [system-boundaries-are-validating-and-evolution-tolerant](system.md#system-boundaries-are-validating-and-evolution-tolerant)

### unified-system-is-a-closed-self-maintaining-architecture
**Status:** OUT

The system forms a closed self-maintaining belief architecture: end-to-end integrity ensures no operation corrupts consistency, while revision completeness ensures any valid belief configuration is reachable — together guaranteeing the system can evolve to any target state while preserving all invariants — only when all known defects and fragilities are resolved.

**Depends on:** [revision-completeness-follows-from-minimality](revision.md#revision-completeness-follows-from-minimality), [unified-system-maintains-end-to-end-integrity](system.md#unified-system-maintains-end-to-end-integrity)

### unified-system-maintains-end-to-end-integrity
**Status:** OUT

The fully unified system — minimal primitives, sound multi-agent scaling, and complete dialectical revision — also maintains end-to-end integrity across all internal and external operations, meaning the design's unification extends from semantic minimality through operational safety to produce a system where every component inherits both the expressiveness and the integrity guarantees of the core.

**Depends on:** [internal-and-external-integrity-are-unified](external.md#internal-and-external-integrity-are-unified), [system-is-unified-minimal-dialectical-and-scalable](system.md#system-is-unified-minimal-dialectical-and-scalable)
**Supports:** [integrity-is-an-emergent-consequence-of-minimality](other.md#integrity-is-an-emergent-consequence-of-minimality), [unified-system-is-a-closed-self-maintaining-architecture](system.md#unified-system-is-a-closed-self-maintaining-architecture)
