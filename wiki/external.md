# external

[Back to index](index.md)

### all-external-belief-paths-are-safely-bounded
**Status:** OUT

Both external belief integration pathways — LLM-driven derivation (bounded by fail-soft validation, Jaccard retraction guards, environment isolation, and safe propagation) and multi-agent import (bounded by namespace isolation, reversible kill-switches, and layered architecture) — are defense-in-depth pipelines that cannot corrupt the host network.

**Depends on:** [llm-mutations-are-bounded-end-to-end](llm.md#llm-mutations-are-bounded-end-to-end), [multi-agent-safety-spans-all-layers](agent.md#multi-agent-safety-spans-all-layers)
**Supports:** [internal-and-external-integrity-are-unified](external.md#internal-and-external-integrity-are-unified), [operational-safety-spans-all-mutation-sources](spans.md#operational-safety-spans-all-mutation-sources)

### all-external-execution-is-subprocess-isolated
**Status:** IN

All LLM-facing operations execute through subprocess isolation with environment scrubbing: derive shells out to CLI binaries rather than importing SDKs (achieving provider agnosticism), and both derive and ask independently strip the CLAUDECODE environment variable (preventing recursive invocation) — two independent safety goals achieved through the same architectural choice.

**Depends on:** [derive-uses-subprocess-not-sdk](derive.md#derive-uses-subprocess-not-sdk), [llm-subprocess-isolation-prevents-recursion](llm.md#llm-subprocess-isolation-prevents-recursion)
**Supports:** [llm-integration-is-defense-in-depth-across-layers](llm.md#llm-integration-is-defense-in-depth-across-layers)

### all-external-inputs-produce-correct-state
**Status:** OUT

Both external input pathways (LLM derivation and multi-agent import) produce a fully correct persisted network state — bounded validation prevents invalid beliefs, namespace isolation prevents cross-contamination, and layered reconciliation handles all truth states — provided the agent count bug is fixed and missing source files are detected.

**Depends on:** [llm-mutations-are-bounded-end-to-end](llm.md#llm-mutations-are-bounded-end-to-end), [multi-agent-safety-spans-all-layers](agent.md#multi-agent-safety-spans-all-layers)

### all-external-inputs-safely-integrated
**Status:** IN

Both LLM-derived beliefs and agent-imported beliefs are safely integrated into the network: defensive validation with retraction guards bounds LLM output, while complete reconciliation with dual modes and heterogeneous truth handling manages agent imports.

**Depends on:** [import-provides-complete-reconciliation](import.md#import-provides-complete-reconciliation), [llm-driven-mutations-are-safely-bounded](llm.md#llm-driven-mutations-are-safely-bounded)
**Supports:** [external-belief-management-is-complete](external.md#external-belief-management-is-complete)

### ask-degrades-across-all-external-dependencies
**Status:** OUT

The ask module degrades gracefully across all three external dependencies: LLM binary (catches TimeoutExpired/RuntimeError, falls back to raw search), FTS5 index (self-healing derived index with substring fallback), and source chunks database (catches OperationalError/DatabaseError and returns empty results) — no single external system failure prevents useful query responses.

**Depends on:** [ask-is-fault-tolerant-and-bounded](other.md#ask-is-fault-tolerant-and-bounded), [ask-sources-db-failure-silently-degrades](other.md#ask-sources-db-failure-silently-degrades)

### external-belief-ingestion-is-defensively-layered
**Status:** OUT

External beliefs enter the system through defensively-layered pipelines regardless of source: LLM derivation applies fail-soft validation, Jaccard retraction guards, and environment isolation, while agent import provides dual reconciliation modes with heterogeneous truth state handling — both converge on the same underlying mutation infrastructure.

**Depends on:** [derive-pipeline-is-defensive](derive.md#derive-pipeline-is-defensive), [import-sync-has-dual-reconciliation-modes](import.md#import-sync-has-dual-reconciliation-modes)
**Supports:** [external-beliefs-defensively-contained](external.md#external-beliefs-defensively-contained), [external-ingestion-is-format-resilient-and-defensively-layered](external.md#external-ingestion-is-format-resilient-and-defensively-layered)

### external-belief-lifecycle-is-complete
**Status:** IN

The system manages external beliefs across their full lifecycle: import/sync provides dual reconciliation modes with heterogeneous truth state handling and namespace auto-wiring, while staleness checking detects source drift for CI gating — beliefs are tracked from initial ingestion through ongoing validity monitoring.

**Depends on:** [import-provides-complete-reconciliation](import.md#import-provides-complete-reconciliation), [staleness-is-conservative-ci-gate](other.md#staleness-is-conservative-ci-gate)
**Supports:** [external-beliefs-are-defended-and-lifecycle-managed](external.md#external-beliefs-are-defended-and-lifecycle-managed), [external-lifecycle-is-complete-and-automatically-maintained](external.md#external-lifecycle-is-complete-and-automatically-maintained)

### external-belief-management-is-complete
**Status:** OUT

External beliefs are managed across their entire lifecycle with no gap between any management phase: safely integrated through defensive validation pipelines, lifecycle-managed across dual import/sync reconciliation modes, and actively kept current through staleness detection and derive pipeline refresh — providing complete external belief management from ingestion through retirement.

**Depends on:** [all-external-inputs-safely-integrated](external.md#all-external-inputs-safely-integrated), [external-beliefs-are-safe-and-current](external.md#external-beliefs-are-safe-and-current)
**Supports:** [external-beliefs-achieve-integration-parity](external.md#external-beliefs-achieve-integration-parity)

### external-beliefs-achieve-integration-parity
**Status:** OUT

External beliefs achieve full parity with internal beliefs: they are managed across their complete lifecycle with no gap between any management phase AND participate in the same deterministic revision engine as all other belief origins — external provenance is a property of ingestion, not of ongoing maintenance.

**Depends on:** [all-belief-origins-share-deterministic-revision](deterministic.md#all-belief-origins-share-deterministic-revision), [external-belief-management-is-complete](external.md#external-belief-management-is-complete)
**Supports:** [closed-loop-is-origin-agnostic](other.md#closed-loop-is-origin-agnostic), [deterministic-history-extends-to-all-origins](deterministic.md#deterministic-history-extends-to-all-origins), [external-beliefs-are-invariant-equivalent](external.md#external-beliefs-are-invariant-equivalent)

### external-beliefs-achieve-total-integration
**Status:** OUT

External beliefs achieve total integration along all quality axes simultaneously: grounded across every invariant dimension (origin, time, structure), participating in all correction mechanisms with full auditability, and fully equivalent to internal beliefs — no property distinguishes external from internal.

**Depends on:** [external-beliefs-are-correctable-and-invariant-equivalent](external.md#external-beliefs-are-correctable-and-invariant-equivalent), [external-beliefs-are-fully-invariant-grounded](external.md#external-beliefs-are-fully-invariant-grounded)
**Supports:** [total-invariant-preservation-encompasses-all-beliefs](beliefs.md#total-invariant-preservation-encompasses-all-beliefs)

### external-beliefs-are-correctable-and-invariant-equivalent
**Status:** OUT

External beliefs achieve full parity along two independent quality axes: invariant equivalence ensures they participate in the same consistency guarantees as internal beliefs, while origin-spanning auditability ensures they are fully correctable with complete traceable history.

**Depends on:** [corrections-span-all-origins-with-full-auditability](other.md#corrections-span-all-origins-with-full-auditability), [external-beliefs-are-invariant-equivalent](external.md#external-beliefs-are-invariant-equivalent)
**Supports:** [external-beliefs-achieve-total-integration](external.md#external-beliefs-achieve-total-integration), [self-sustaining-preservation-encompasses-external-beliefs](self.md#self-sustaining-preservation-encompasses-external-beliefs)

### external-beliefs-are-defended-and-lifecycle-managed
**Status:** OUT

External beliefs are managed end-to-end across a complete trust boundary: defensively contained at ingestion through layered validation pipelines and namespace isolation, then actively lifecycle-managed through dual reconciliation modes, staleness detection against source material, and CI gating — no phase of external belief existence lacks oversight.

**Depends on:** [external-belief-lifecycle-is-complete](external.md#external-belief-lifecycle-is-complete), [external-beliefs-defensively-contained](external.md#external-beliefs-defensively-contained)
**Supports:** [external-beliefs-are-safe-and-current](external.md#external-beliefs-are-safe-and-current), [external-integration-is-architecturally-safe](external.md#external-integration-is-architecturally-safe)

### external-beliefs-are-fully-invariant-grounded
**Status:** OUT

External beliefs achieve complete invariant equivalence with internal beliefs, and those invariants are anchored along all three dimensions — origin, temporal, and structural — meaning external integration participates in the system's deepest invariant guarantees, not merely surface-level safety.

**Depends on:** [external-beliefs-are-invariant-equivalent](external.md#external-beliefs-are-invariant-equivalent), [invariants-are-origin-time-and-structurally-grounded](other.md#invariants-are-origin-time-and-structurally-grounded)
**Supports:** [external-beliefs-achieve-total-integration](external.md#external-beliefs-achieve-total-integration), [origin-agnostic-loop-grounds-external-invariants](external.md#origin-agnostic-loop-grounds-external-invariants)

### external-beliefs-are-invariant-equivalent
**Status:** OUT

External beliefs achieve complete equivalence with internal beliefs at every system level: they participate in identical invariant-preserving revision systems and achieve full behavioral integration parity — the system provides no mechanism to distinguish external from internal beliefs in terms of protection or management.

**Depends on:** [external-beliefs-achieve-integration-parity](external.md#external-beliefs-achieve-integration-parity), [external-integration-preserves-all-invariants](external.md#external-integration-preserves-all-invariants)
**Supports:** [external-beliefs-are-correctable-and-invariant-equivalent](external.md#external-beliefs-are-correctable-and-invariant-equivalent), [external-beliefs-are-fully-invariant-grounded](external.md#external-beliefs-are-fully-invariant-grounded)

### external-beliefs-are-safe-and-current
**Status:** OUT

External beliefs are managed end-to-end across their complete lifecycle with no gap between ingestion safety and ongoing maintenance: defensively contained at ingestion through layered validation, correctly lifecycle-managed through import reconciliation and staleness checking, and actively tracked for currency — no external belief enters unvalidated, drifts undetected, or persists without lifecycle oversight.

**Depends on:** [belief-currency-is-actively-managed](other.md#belief-currency-is-actively-managed), [external-beliefs-are-defended-and-lifecycle-managed](external.md#external-beliefs-are-defended-and-lifecycle-managed)
**Supports:** [external-belief-management-is-complete](external.md#external-belief-management-is-complete)

### external-beliefs-defensively-contained
**Status:** OUT

External beliefs pass through two independent safety layers: defensive ingestion pipelines (fail-soft validation, Jaccard guards, dual import/sync reconciliation modes) filter and validate beliefs on entry, while the self-contained agent subsystem (namespace isolation, relay-pair kill-switches, reversible lifecycle management) constrains their operational footprint after ingestion.

**Depends on:** [agent-subsystem-is-self-contained](agent.md#agent-subsystem-is-self-contained), [external-belief-ingestion-is-defensively-layered](external.md#external-belief-ingestion-is-defensively-layered)
**Supports:** [external-beliefs-are-defended-and-lifecycle-managed](external.md#external-beliefs-are-defended-and-lifecycle-managed), [external-inputs-face-defense-in-depth](external.md#external-inputs-face-defense-in-depth), [external-surface-is-fully-controlled](external.md#external-surface-is-fully-controlled), [revision-safety-spans-internal-and-external](revision.md#revision-safety-spans-internal-and-external), [trust-boundary-is-architecturally-enforced](other.md#trust-boundary-is-architecturally-enforced)

### external-deps-become-premises
**Status:** IN

A claim whose `depends_on` references are all absent from the beliefs file gets no justifications and is added as a premise (IN by default).


### external-ingestion-is-format-resilient-and-defensively-layered
**Status:** OUT

External belief ingestion achieves both defensive containment (fail-soft validation, Jaccard retraction guards, dual import/sync reconciliation, namespace isolation) AND format resilience (parser version fallback, forward-compatible metadata parsing, prose-tolerant JSON extraction), ensuring robust integration even as external source formats evolve unpredictably.

**Depends on:** [external-belief-ingestion-is-defensively-layered](external.md#external-belief-ingestion-is-defensively-layered), [format-resilience-spans-all-external-interfaces](spans.md#format-resilience-spans-all-external-interfaces)
**Supports:** [external-ingestion-is-resilient-and-convergent](external.md#external-ingestion-is-resilient-and-convergent)

### external-ingestion-is-resilient-and-convergent
**Status:** OUT

External belief ingestion achieves end-to-end reliability through two complementary properties: format resilience absorbs syntactic variation at the parsing boundary (dual parser versions, schema migration tolerance, prose-tolerant JSON extraction with defensive layering), while deterministic convergence ensures consistent final state through fixpoint reconciliation regardless of input ordering or repeated application.

**Depends on:** [all-reconciliation-converges-deterministically](other.md#all-reconciliation-converges-deterministically), [external-ingestion-is-format-resilient-and-defensively-layered](external.md#external-ingestion-is-format-resilient-and-defensively-layered)

### external-inputs-face-defense-in-depth
**Status:** OUT

External beliefs face defense in depth across two independent containment layers: input-level containment (defensive validation pipelines, agent namespace isolation) prevents bad data from entering, while system-level containment (architectural layer boundaries, lifecycle-aware checking and propagation) prevents bad data from persisting or spreading.

**Depends on:** [external-beliefs-defensively-contained](external.md#external-beliefs-defensively-contained), [integrity-enforced-across-architecture-and-lifecycle](lifecycle.md#integrity-enforced-across-architecture-and-lifecycle)
**Supports:** [all-belief-modification-paths-are-operationally-safe](safe.md#all-belief-modification-paths-are-operationally-safe)

### external-integration-is-architecturally-safe
**Status:** OUT

External beliefs are end-to-end safe within the system's architecture: defensively contained at ingestion and lifecycle-managed thereafter (external belief thread) within the same three-layer boundaries and atomic mutation guarantees that protect internal operations (architecture thread).

**Depends on:** [architecture-enforces-structural-and-operational-safety](other.md#architecture-enforces-structural-and-operational-safety), [external-beliefs-are-defended-and-lifecycle-managed](external.md#external-beliefs-are-defended-and-lifecycle-managed)
**Supports:** [external-integration-preserves-all-invariants](external.md#external-integration-preserves-all-invariants)

### external-integration-is-hardened-and-boundary-controlled
**Status:** OUT

LLM integration achieves production-grade robustness (bounded execution, fail-soft error handling, process isolation, fault tolerance) while information boundaries are controlled at every level (authorization gating, budget constraints, defensive ingestion) — the system neither leaks sensitive information outward nor admits unvalidated beliefs inward.

**Depends on:** [information-boundaries-are-controlled-at-all-levels](other.md#information-boundaries-are-controlled-at-all-levels), [llm-integration-is-production-hardened](llm.md#llm-integration-is-production-hardened)
**Supports:** [information-flow-is-controlled-in-both-directions](other.md#information-flow-is-controlled-in-both-directions)

### external-integration-preserves-all-invariants
**Status:** OUT

External beliefs are architecturally safe at ingestion and participate in the same invariant-preserving revision system as all other belief origins — architectural containment and revision parity together ensure external integration cannot corrupt system invariants.

**Depends on:** [external-integration-is-architecturally-safe](external.md#external-integration-is-architecturally-safe), [revision-invariants-span-all-origins](revision.md#revision-invariants-span-all-origins)
**Supports:** [external-beliefs-are-invariant-equivalent](external.md#external-beliefs-are-invariant-equivalent)

### external-interface-is-bidirectionally-bounded
**Status:** OUT

The system's interaction with external systems is bounded in both directions: output is budget-limited through accurate token estimation ensuring context windows are respected, and input drift is comprehensively detected through staleness checking — no unbounded data flows cross the system boundary.

**Depends on:** [compact-budget-controls-output-size](compact.md#compact-budget-controls-output-size), [staleness-checking-is-comprehensive](other.md#staleness-checking-is-comprehensive)
**Supports:** [external-surface-is-fully-controlled](external.md#external-surface-is-fully-controlled), [system-reliability-spans-internal-and-external](system.md#system-reliability-spans-internal-and-external)

### external-lifecycle-achieves-topology-accurate-quality
**Status:** OUT

External beliefs receive quality-complete self-correction that operates on accurate convergent topology — every correction to an externally-originated belief propagates through verified dependency structure with complete fidelity, ensuring external belief quality is maintained with the same topological accuracy as internal beliefs.

**Depends on:** [external-lifecycle-receives-quality-complete-self-correction](external.md#external-lifecycle-receives-quality-complete-self-correction), [topology-accurate-self-correction-is-quality-complete](topology.md#topology-accurate-self-correction-is-quality-complete)

### external-lifecycle-is-complete-and-automatically-maintained
**Status:** IN

External beliefs achieve both complete lifecycle management (dual reconciliation modes with heterogeneous truth state handling and staleness detection) and automated ongoing maintenance (idempotent sync with cascade preservation and full source staleness coverage), enabling zero-touch external belief management

**Depends on:** [automated-sync-achieves-full-lifecycle-coverage](lifecycle.md#automated-sync-achieves-full-lifecycle-coverage), [external-belief-lifecycle-is-complete](external.md#external-belief-lifecycle-is-complete)
**Supports:** [external-lifecycle-operates-within-rich-governance](external.md#external-lifecycle-operates-within-rich-governance)

### external-lifecycle-is-deterministic-and-trust-bounded
**Status:** OUT

External beliefs follow a fully deterministic lifecycle from ingestion through ongoing maintenance, enclosed within verified trust boundaries at every phase: structural verification with trust-bounded bidirectional flow control at the perimeter, and deterministic architecturally-grounded lifecycle management governing internal state trajectories.

**Depends on:** [external-surface-is-verified-trust-bounded-and-bidirectionally-controlled](external.md#external-surface-is-verified-trust-bounded-and-bidirectionally-controlled), [lifecycle-is-deterministic-grounded-and-structurally-sound](lifecycle.md#lifecycle-is-deterministic-grounded-and-structurally-sound)
**Supports:** [external-lifecycle-receives-quality-complete-self-correction](external.md#external-lifecycle-receives-quality-complete-self-correction)

### external-lifecycle-operates-within-rich-governance
**Status:** IN

External beliefs achieve complete automated lifecycle management within a metadata-enabled source-grounded governance framework — creation, reconciliation, staleness detection, and ongoing maintenance all operate under rich governance extending beyond binary truth values to structured lifecycle state.

**Depends on:** [external-lifecycle-is-complete-and-automatically-maintained](external.md#external-lifecycle-is-complete-and-automatically-maintained), [lifecycle-governance-is-metadata-enabled-and-source-grounded](lifecycle.md#lifecycle-governance-is-metadata-enabled-and-source-grounded)

### external-lifecycle-receives-quality-complete-self-correction
**Status:** OUT

External beliefs follow deterministic trust-bounded lifecycles AND receive self-correction that is complete across all quality dimensions — grounded in source integrity, documented with referenceable artifacts, convergent on accurate topology, and evolution-tolerant — ensuring external beliefs are not merely contained but actively maintained at the same quality standard as internal beliefs.

**Depends on:** [external-lifecycle-is-deterministic-and-trust-bounded](external.md#external-lifecycle-is-deterministic-and-trust-bounded), [self-correction-is-complete-across-all-quality-dimensions](self.md#self-correction-is-complete-across-all-quality-dimensions)
**Supports:** [external-lifecycle-achieves-topology-accurate-quality](external.md#external-lifecycle-achieves-topology-accurate-quality)

### external-surface-is-fully-controlled
**Status:** OUT

The system's external surface is fully controlled along independent axes: bidirectional bounds constrain output size (token budgets) and input quality (staleness detection), while defensive containment layers (validation pipelines, namespace isolation) prevent external beliefs from violating internal invariants.

**Depends on:** [external-beliefs-defensively-contained](external.md#external-beliefs-defensively-contained), [external-interface-is-bidirectionally-bounded](external.md#external-interface-is-bidirectionally-bounded)
**Supports:** [information-boundaries-are-controlled-at-all-levels](other.md#information-boundaries-are-controlled-at-all-levels), [system-is-externally-controlled-and-internally-self-correcting](system.md#system-is-externally-controlled-and-internally-self-correcting)

### external-surface-is-verified-and-trust-bounded
**Status:** OUT

The complete external-facing surface achieves both structural verification (pure delegation with hermetic tests and fault tolerance across all information paths) and comprehensive trust enforcement (architectural self-containment with information flow control at every boundary) — no external interaction bypasses either the verification chain or trust controls.

**Depends on:** [trust-and-information-boundaries-are-comprehensively-enforced](other.md#trust-and-information-boundaries-are-comprehensively-enforced), [user-interface-is-verified-and-fault-tolerant](other.md#user-interface-is-verified-and-fault-tolerant)
**Supports:** [external-surface-is-verified-trust-bounded-and-bidirectionally-controlled](external.md#external-surface-is-verified-trust-bounded-and-bidirectionally-controlled)

### external-surface-is-verified-trust-bounded-and-bidirectionally-controlled
**Status:** OUT

The system's complete external perimeter achieves defense-in-depth through three independently-established properties: structural verification (pure delegation with hermetic tests), trust boundaries (self-containment and defensive ingestion), and bidirectional flow control (token budgets constraining output, hardened LLM integration constraining input) — no external interaction bypasses all three layers.

**Depends on:** [external-surface-is-verified-and-trust-bounded](external.md#external-surface-is-verified-and-trust-bounded), [verified-interface-controls-bidirectional-flow](other.md#verified-interface-controls-bidirectional-flow)
**Supports:** [external-lifecycle-is-deterministic-and-trust-bounded](external.md#external-lifecycle-is-deterministic-and-trust-bounded)

### internal-and-external-integrity-are-unified
**Status:** OUT

Every operation — internal mutations (atomic transactions, deterministic propagation) and external belief ingestion (defensive validation, environment isolation, agent containment) — maintains end-to-end integrity through complementary safety mechanisms.

**Depends on:** [all-external-belief-paths-are-safely-bounded](external.md#all-external-belief-paths-are-safely-bounded), [operational-integrity-is-end-to-end](other.md#operational-integrity-is-end-to-end)
**Supports:** [complete-unified-system-is-production-ready](complete.md#complete-unified-system-is-production-ready), [integrity-and-scalability-are-complementary](other.md#integrity-and-scalability-are-complementary), [integrity-is-boundary-and-source-agnostic](source.md#integrity-is-boundary-and-source-agnostic), [unified-system-maintains-end-to-end-integrity](system.md#unified-system-maintains-end-to-end-integrity)

### network-has-zero-external-dependencies
**Status:** IN

The Network class imports only stdlib (`collections.deque`, `datetime`) plus package data types (`Node`, `Justification`, `Nogood`); it has zero external dependencies.

**Supports:** [project-has-zero-external-coupling](external.md#project-has-zero-external-coupling)

### network-no-external-deps
**Status:** IN

`network.py` depends only on stdlib (`deque`, `datetime`) and project dataclasses; it has zero external package dependencies.


### origin-agnostic-loop-grounds-external-invariants
**Status:** OUT

External beliefs achieve complete invariant grounding along all three dimensions (origin, time, structure) precisely because the closed maintenance loop operates origin-agnostically — the loop's indifference to belief provenance is the mechanism that delivers invariant parity to external beliefs.

**Depends on:** [closed-loop-is-origin-agnostic](other.md#closed-loop-is-origin-agnostic), [external-beliefs-are-fully-invariant-grounded](external.md#external-beliefs-are-fully-invariant-grounded)
**Supports:** [origin-agnosticism-unifies-trustworthiness-and-grounding](other.md#origin-agnosticism-unifies-trustworthiness-and-grounding)

### project-has-zero-external-coupling
**Status:** IN

The project enforces zero external coupling at both the packaging level (empty `dependencies` list in pyproject.toml) and the implementation level (core Network class uses only stdlib imports), eliminating supply-chain risk and simplifying deployment.

**Depends on:** [network-has-zero-external-dependencies](external.md#network-has-zero-external-dependencies), [zero-runtime-dependencies](other.md#zero-runtime-dependencies)
**Supports:** [architecture-is-self-contained-and-safely-layered](self.md#architecture-is-self-contained-and-safely-layered), [system-resource-footprint-is-minimal-at-all-phases](system.md#system-resource-footprint-is-minimal-at-all-phases)
