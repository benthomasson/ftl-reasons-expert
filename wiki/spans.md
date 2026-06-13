# spans

[Back to index](index.md)

The existing page already covers all 16 beliefs accurately and is well-written. The content matches the current belief statuses (3 IN, 13 OUT) and the dependency relationships. I don't see anything that needs updating — the page is comprehensive and correctly structured.

Here's the content (unchanged, as it's already correct):

A **span** in the belief network is a derived property asserting that some guarantee — safety, determinism, efficiency, or consistency — holds uniformly across multiple architectural layers, lifecycle phases, or system dimensions. Spans are synthesis beliefs: they combine two or more independently-established properties to claim that no gap exists in coverage. Most span beliefs are currently retracted (OUT), reflecting either changes in their supporting beliefs or evolving understanding of the system's guarantees.

## Currently Held Spans

Three span beliefs remain IN, representing the system's strongest cross-cutting claims.

### Belief Consistency

The system maintains belief consistency through two complementary mechanisms that target different quality dimensions (belief-consistency-spans-structural-and-semantic-dimensions). Deduplication resolves *structural* redundancy — merging equivalent beliefs while preserving network topology through user-editable, auditable plans. Contradiction management resolves *semantic* inconsistency through dependency-directed backtracking with consistent nogood IDs and minimal-disruption culprit selection. This belief depends on both deduplication being topology-preserving and auditable ([dedup-is-topology-preserving-and-auditable](topology.md#dedup-is-topology-preserving-and-auditable)) and contradiction management being complete and traceable ([contradiction-management-is-complete-and-traceable](complete.md#contradiction-management-is-complete-and-traceable)).

### Determinism Through Source Integrity

Determinism is maintained end-to-end from belief revision semantics through source integrity verification (determinism-spans-revision-semantics-through-source-integrity). Revision operations produce deterministic, traceable state transitions with structured before/after diffs and controlled irreversibility, and the source verification that triggers those revisions is itself deterministic and architecturally grounded — fail-safe path resolution, exact SHA-256 comparison, clean layer boundaries. No step in the revision-triggered-by-source-change pathway introduces non-determinism. This span bridges [revision-semantics-are-deterministic-and-traceable](revision.md#revision-semantics-are-deterministic-and-traceable) with [source-integrity-is-deterministic-and-architecturally-grounded](source.md#source-integrity-is-deterministic-and-architecturally-grounded), and itself supports a broader governance claim ([rich-governance-determinism-spans-revision-through-source](governance.md#rich-governance-determinism-spans-revision-through-source)).

### Exception Safety

Exception safety spans both the TMS core and the source lifecycle pipeline (exception-safety-spans-tms-and-source-lifecycle). The TMS core resolves exceptional conditions — contradictions via backtracking, challenges via deterministic propagation — while the source integrity pipeline handles its own failure modes: missing files, changed content, deleted sources. The claim is that no exception in either domain can produce inconsistent or undefined state. This span supports both [rich-governance-is-exception-safe](governance.md#rich-governance-is-exception-safe) and [source-to-tms-integrity-is-deterministic-and-exception-safe](source.md#source-to-tms-integrity-is-deterministic-and-exception-safe).

## Retracted Spans

The remaining span beliefs are OUT, meaning their supporting claims have been retracted or revised. They document the system's *intended* cross-cutting guarantees and may be restored as supporting beliefs are re-established.

### Safety and Integrity

Several retracted spans addressed safety across architectural layers and mutation sources. Data integrity was claimed to be enforced across all three architectural layers — clean boundaries, snapshot persistence, and conservative staleness checking (data-integrity-spans-architecture). Mutation safety was claimed along three orthogonal dimensions simultaneously: source, semantic, and lifecycle (mutation-safety-spans-all-dimensions). Operational safety was asserted across both internal mutation pipelines and external belief ingestion pathways (operational-safety-spans-all-mutation-sources). Edge case safety was claimed across both temporal dimensions — creation time and maintenance time (edge-case-safety-spans-creation-and-maintenance).

### Quality and Fault Tolerance

Dual quality enforcement spanned automated self-correction and explicit quality review as independent, non-interfering mechanisms (dual-quality-enforcement-spans-automated-and-explicit). Fault tolerance was claimed to cover the complete belief quality spectrum from passive inspection through active self-correction, with the system maintaining quality assurance autonomously even when LLMs or external resources are unavailable (fault-tolerance-spans-inspection-through-self-correction).

### Defense in Depth and Format Resilience

Defense in depth was asserted at every external interface through two layers: LLM integration defenses (bounded execution, fail-soft handling, subprocess isolation) and system boundary enforcement spanning validation, evolution tolerance, and resource constraints (defense-in-depth-spans-llm-and-system-boundaries). Format resilience was claimed across all external interfaces, from derive output parsers through import parsers to LLM response parsing with regex extraction fallback (format-resilience-spans-all-external-interfaces).

### Resource Efficiency

Two overlapping span beliefs addressed resource efficiency across the operational pipeline. One claimed efficiency from static footprint through runtime budgets (resource-efficiency-spans-footprint-through-budgets), while the other claimed it across the complete pipeline from packaging through derivation to output generation (resource-efficiency-spans-full-pipeline). The latter supports four downstream beliefs including [defense-in-depth-is-resource-efficient](other.md#defense-in-depth-is-resource-efficient) and [self-correction-operates-within-efficient-pipeline](self.md#self-correction-operates-within-efficient-pipeline).

### Dispute Resolution and Production Correctness

Dispute resolution was claimed to span all belief origins, with both dialectical and automated contradiction mechanisms applying uniformly through identical deterministic revision pipelines (dispute-resolution-spans-all-origins). Verified production correctness was asserted across all origins, combining verified architecture correctness with deterministic state trajectories extending to externally-originated beliefs at full integration parity (verified-production-correctness-spans-all-origins). The operational profile — safety, assurance, and resource-boundedness — was claimed to hold identically across all storage backends including both SQLite and PostgreSQL (operational-profile-spans-all-backends).

### Minimality as Generative Root

Minimality was asserted as the shared generative root of both forward and backward system properties (minimality-spans-computation-and-revision): forward computation achieves uniformity and determinism, backward revision achieves universal safety. This claim — that the same minimal primitives produce correctness in both directions without separate design efforts — supported the broader claim that minimality sustains closed-loop maintenance.

## Structural Patterns

Spans share a common structure: they combine two or more independently-justified properties to assert gap-free coverage. The naming convention `X-spans-Y-and-Z` or `X-spans-all-Y` signals this synthesis role. Because spans sit high in the dependency graph, they are sensitive to retraction cascades — when any supporting belief goes OUT, the span goes OUT with it. The concentration of OUT spans (13 of 16) reflects this fragility: these are the first beliefs to fall and the last to be restored.

The three surviving spans — consistency, determinism, and exception safety — represent the system's most robustly supported cross-cutting guarantees. Their persistence suggests that the structural/semantic consistency mechanisms, the revision-through-source determinism chain, and the TMS/source exception handling are the most stable parts of the architecture.
