# governance

[Back to index](index.md)

The ftl-reasons system implements a comprehensive governance framework that extends well beyond binary truth maintenance. Rather than simply tracking whether beliefs are IN or OUT, governance encompasses the full lifecycle of belief state — including retraction reasons, staleness tracking, access control, and metadata propagation — ensuring that every modification produces predictable, recoverable, and verifiable outcomes across the entire dependency graph.

## Rich Governance Beyond Binary Truth

At its foundation, governance in ftl-reasons manages state richer than binary truth values. The deterministic lifecycle-complete architecture ensures predictable state trajectories and gapless lifecycle monitoring so that no belief escapes management, while metadata-enabled governance extends these guarantees to retraction reasons, staleness tracking, and access control (`rich-governance-is-deterministic-and-lifecycle-complete`). This rich governance operates identically across all storage backends, achieving full behavioral parity between SQLite and PostgreSQL (`rich-governance-spans-all-backends`).

Importantly, this rich state governance is not layered on top of the truth maintenance system as additional machinery. While an earlier belief held that rich governance emerged from the same minimal foundations that unify truth computation and belief revision (`rich-governance-emerges-from-minimal-foundations`, OUT), the current understanding frames the relationship through a more nuanced chain: rich governance inherits its determinism from the unified minimality-completeness-determinism triad, meaning predictable state trajectories and gapless monitoring arise from the same minimal primitives that produce comprehensive truth maintenance coverage (`rich-governance-inherits-minimality-completeness-determinism-unity`).

## Three Dimensions of Completeness

Governance achieves completeness along three independent output dimensions simultaneously (`governance-is-topology-source-and-traceability-complete`):

- **Topology completeness** — every state change propagates to all transitively dependent nodes, including outlist-connected paths, even in the presence of dangling references (`metadata-governance-has-topology-complete-propagation`). Metadata-carried lifecycle state — retraction flags, stale reasons, access tags — actively governs truth propagation through this topology-complete, inconsistency-safe infrastructure (`metadata-governance-flows-through-safe-topology`).

- **Source completeness** — lifecycle governance achieves gap-free source coverage, ensuring every source file is tracked with no silent gaps (`governance-achieves-topology-and-source-completeness`).

- **Traceability completeness** — governance actions produce metadata-enriched state transitions, so every modification is not only propagated and source-verified but also traceable through rich lifecycle metadata (`[topology-complete-governance-produces-rich-traceable-state](topology.md#topology-complete-governance-produces-rich-traceable-state)`, referenced by `governance-is-topology-source-and-traceability-complete`).

This three-dimensional completeness means every governance action produces a verifiable, traceable, source-grounded state transition.

## Determinism and Its Origins

Governance determinism has a distinctive provenance: it is not independently engineered but emerges as a consequence of the system's minimal foundations (`governance-determinism-is-minimality-emergent-through-source`). The same minimality that produces governance completeness also produces its end-to-end deterministic reach, spanning from belief revision semantics through source integrity verification (`rich-governance-determinism-spans-revision-through-source`).

This determinism has a complete lifecycle — it is generated through source integrity as an emergent consequence of minimality, and simultaneously preserved through exception safety underpinned by evaluation purity (`governance-determinism-is-generated-and-preserved`). Evaluation purity provides the semantic foundation for richly governed dialectics, while exception safety preserves that governance through all failure modes (`evaluation-purity-grounds-governance-that-exception-safety-preserves`). The two roles are complementary: purity creates the governance properties, exception safety ensures they are never lost.

## Exception Safety

Rich lifecycle governance operates within an exception-safe framework spanning both the TMS core and source lifecycle (`rich-governance-is-exception-safe`). Contradictions are resolved through deterministic backtracking, challenges reach correct truth states through crash-safe propagation, and source failures degrade gracefully through fail-safe path resolution. This ensures that rich state transitions never produce corrupted or inconsistent lifecycle metadata, even under exceptional conditions.

Combined with determinism and source grounding, exception safety forms part of the governance triad: the system is simultaneously predictable in its state trajectories and resilient to all exceptional conditions (`rich-governance-is-deterministic-exception-safe-and-source-grounded`).

## Dual Independent Grounding

The governance framework receives assurance from two fully independent grounding chains (`governance-has-dual-independent-grounding-chains`): dialectical operations are grounded by evaluation purity and semantic uniformity, while the rich governance framework itself is grounded by determinism, exception safety, and source integrity — five independent assurance dimensions from orthogonal chains.

This dual grounding means that no single semantic foundation failure can undermine the governance framework's three-dimensional completeness guarantees (`governance-completeness-is-dually-grounded`). The three-dimensional completeness rests on two independent chains, providing resilience against foundation-level failures.

## Bidirectional Modifications and the Governance Triad

Governance applies uniformly to modifications in both directions. Bidirectional belief modifications — contradiction resolution through traceable backtracking and defeat reversal with guided recovery — simultaneously achieve all three governance dimensions: rich revision governance extending beyond binary truth, exception-safe recoverability across all failure modes, and topology-complete metadata propagation through the full dependency graph (`bidirectional-modifications-achieve-full-governance-triad`).

This means every belief modification, whether forward or backward, achieves full governance assurance (`all-modifications-achieve-governance-assured-topology-completeness`). The governance assurance itself rests on dual independent grounding chains, so modifications are universally governed AND that governance is independently grounded by both evaluation purity and edge-case uniformity (`all-modifications-are-dually-grounded-governance-assured`).

## Dialectical Assurance

Dialectical operations — challenge, defend, and reversal — achieve complete bidirectional assurance within the governance framework (`dialectical-assurance-achieves-governance-completeness`). Every dialectical operation produces outcomes that meet the full governance quality bar across all three output dimensions (topology, source, and traceability).

Governance completeness is simultaneously dialectically assured and independently grounded (`governance-is-dialectically-assured-and-dually-grounded`), achieving both operational confidence through bidirectional dialectics and epistemic independence through dual evaluation chains.

## Self-Reinforcing Universality

At its highest level of abstraction, the governance framework is self-reinforcing: every belief modification achieves governance assurance that is itself dialectically complete (`all-modifications-achieve-dialectically-assured-governance`). Modifications operate within dually-grounded governance backed by complete bidirectional dialectical assurance, meaning every modification is simultaneously topology-complete, dually-grounded, and dialectically assured.

The governance framework constraining all modifications is verified by the same dialectical and grounding mechanisms it employs (`governance-assurance-is-universal-and-self-reinforcing`), creating a self-reinforcing quality guarantee. The framework does not merely govern — it governs its own governance, closing the assurance loop.

## Pending Verification

Several governance-related beliefs remain retracted (OUT), representing verification gaps that have not yet been closed:

- **Reference integrity verification** — the claim that governance and dialectics achieve verified reference integrity at all node ID boundaries (`governance-and-dialectics-have-verified-references`, OUT) and that topology-complete transitions have fully verified references (`governance-topology-is-reference-verified`, OUT) await confirmation from a reference validation audit (issue #126).

- **Evolution tolerance** — the claim that rich governance achieves verified evolution tolerance at all system boundaries (`rich-governance-has-verified-evolution-tolerance`, OUT) depends on an evolution tolerance audit (issue #121) confirming forward-compatibility documentation at all boundaries.

- **Information and output governance** — a cluster of retracted beliefs addresses end-to-end information governance across pipeline and output layers (`information-governance-is-end-to-end-authorized-and-resource-constrained`, `output-governance-is-complete-authorized-and-ci-ready`, `output-governance-is-comprehensive-and-self-sustaining`, all OUT). These represent aspirational properties around authorization, resource constraints, and CI-ready machine-parseable output that have not been fully substantiated.

These retracted beliefs mark the current boundary of verified governance properties, distinguishing established guarantees from properties that remain under active investigation.
