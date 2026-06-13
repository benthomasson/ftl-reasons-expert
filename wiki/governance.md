# governance

[Back to index](index.md)

Governance in the ftl-reasons Truth Maintenance System refers to the framework of properties that ensure every belief modification produces predictable, traceable, and recoverable outcomes. Rather than a single mechanism, governance emerges as a composite of determinism, exception safety, source integrity, and topology-complete propagation — properties that collectively guarantee that the system's state remains well-managed across all operations and failure modes.

## Rich Lifecycle Governance

The governance framework operates on state richer than binary truth values. Where a minimal TMS tracks only whether a belief is IN or OUT, rich lifecycle governance extends to retraction reasons, staleness tracking, and access control metadata (`rich-governance-is-deterministic-and-lifecycle-complete`). This richer state is managed deterministically — given the same inputs, governance produces the same state trajectories — and lifecycle-completely, meaning no belief escapes management regardless of how it was introduced or modified.

This rich governance operates identically across all storage backends, achieving full behavioral parity between SQLite and PostgreSQL for metadata-enabled lifecycle management (`rich-governance-spans-all-backends`).

## Three Dimensions of Completeness

Governance achieves completeness along three independent dimensions simultaneously (`governance-is-topology-source-and-traceability-complete`):

### Topology Completeness

Every state change reaches all transitively dependent nodes in the dependency graph. When a belief's truth value changes, the consequences propagate through the full network — including outlist-connected paths — rather than stopping at direct dependents (`metadata-governance-has-topology-complete-propagation`). The metadata carried in lifecycle state (retraction flags, stale reasons, access tags) actively governs this propagation, and the propagation remains safe even in the presence of dangling references (`metadata-governance-flows-through-safe-topology`).

### Source Completeness

Governance achieves gap-free source coverage, ensuring every source file is tracked with no silent gaps (`governance-achieves-topology-and-source-completeness`). This means integrity verification spans the full pipeline from belief revision semantics through hash-based staleness detection.

### Traceability Completeness

Every governance action produces metadata-enriched state transitions, so outcomes are not merely correct but verifiable and traceable back to their origins. Combined with topology and source completeness, this ensures that every governance action produces a verifiable, source-grounded state transition.

## Exception Safety

Rich governance is resilient to all exceptional conditions (`rich-governance-is-exception-safe`). Exception safety spans both the TMS core and the source lifecycle: contradictions are resolved through deterministic backtracking, challenges reach correct truth states through crash-safe propagation, and source failures degrade gracefully through fail-safe path resolution. The practical consequence is that rich state transitions never produce corrupted or inconsistent lifecycle metadata, even under adversarial conditions.

This exception safety works in tandem with determinism and source grounding — governance is both predictable in its state trajectories and resilient to failure (`rich-governance-is-deterministic-exception-safe-and-source-grounded`).

## Bidirectional Modification Guarantees

Governance applies uniformly to modifications in both directions. Forward operations (topology-complete state transitions) and backward operations (contradiction resolution through backtracking, defeat reversal with guided recovery) both achieve the full governance triad: rich revision governance, exception-safe recoverability, and topology-complete metadata propagation (`bidirectional-modifications-achieve-full-governance-triad`).

This bidirectionality means that every belief modification — regardless of direction — achieves full governance assurance (`all-modifications-achieve-governance-assured-topology-completeness`).

## Dual Independent Grounding

The governance framework does not rest on a single foundation. Instead, it receives assurance from two fully independent grounding chains (`governance-has-dual-independent-grounding-chains`):

1. **Evaluation purity and semantic uniformity** — dialectical operations are grounded by pure evaluation semantics and uniform edge-case handling.
2. **Determinism, exception safety, and source integrity** — the rich governance framework itself is grounded by these three orthogonal properties.

Together, these chains provide five independent assurance dimensions. Because the chains are orthogonal, no single semantic foundation failure can undermine governance's three-dimensional completeness guarantees (`governance-completeness-is-dually-grounded`). Evaluation purity provides the semantic foundation for richly governed dialectics, while exception safety preserves that governance through all failure modes — purity creates the properties and exception safety ensures they are never lost (`evaluation-purity-grounds-governance-that-exception-safety-preserves`).

## Emergent Determinism

A notable architectural property is that governance determinism is not independently engineered. Rather, it emerges as a consequence of the same minimality that generates completeness (`rich-governance-inherits-minimality-completeness-determinism-unity`). The unified minimality-completeness-determinism triad ensures that governance's predictable state trajectories and gapless monitoring arise from the same minimal primitives that produce comprehensive truth maintenance coverage.

This emergent determinism has a complete lifecycle: it is generated through source integrity as a consequence of minimality, and simultaneously preserved through exception safety underpinned by evaluation purity (`governance-determinism-is-generated-and-preserved`).

## Dialectical Assurance

Dialectical operations — challenge, defend, and reversal — achieve complete bidirectional assurance within the governance framework (`dialectical-assurance-achieves-governance-completeness`). Forward reliability and backward recovery are both semantically grounded, and every dialectical operation produces outcomes that meet the full governance quality bar across topology, source, and traceability.

Governance completeness is therefore simultaneously dialectically assured through bidirectional operations and independently grounded through evaluation chains, achieving both operational confidence and epistemic independence (`governance-is-dialectically-assured-and-dually-grounded`).

## Universality and Self-Reinforcement

At the highest level of abstraction, governance assurance is both universal and self-reinforcing (`governance-assurance-is-universal-and-self-reinforcing`). Every belief modification achieves governance assurance that is itself dialectically complete — modifications operate within dually-grounded governance backed by complete bidirectional dialectical assurance (`all-modifications-achieve-dialectically-assured-governance`).

The self-reinforcing property is significant: the governance framework constraining all modifications is verified by the same dialectical and grounding mechanisms it employs. The system does not require external validation of its governance guarantees because those guarantees are established through the same mechanisms that the governance framework itself provides, closing the assurance loop.
