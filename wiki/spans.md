# spans

[Back to index](index.md)



Spans represent a key architectural property of the ftl-reasons system: the ability of quality guarantees to hold *across* subsystem boundaries rather than being confined to a single layer. Where individual beliefs describe properties of specific components — the TMS core, the deduplication engine, the source integrity pipeline — span beliefs capture the insight that these properties compose end-to-end, covering the full pathway from input to output without gaps.

## Belief Consistency Across Structural and Semantic Dimensions

The system maintains belief consistency through two complementary mechanisms, each targeting a different quality dimension (belief-consistency-spans-structural-and-semantic-dimensions). Deduplication resolves *structural* redundancy — merging equivalent beliefs while preserving the network topology, with user-editable, auditable plans. Contradiction management resolves *semantic* inconsistency — using traceable, dependency-directed backtracking with consistent nogood IDs and minimal-disruption culprit selection.

These are not redundant safeguards. Structural redundancy (two nodes expressing the same claim) and semantic inconsistency (two nodes expressing contradictory claims) are distinct failure modes requiring distinct resolution strategies. The span captures the fact that the system addresses both, leaving no category of inconsistency unhandled.

## Determinism from Revision Semantics Through Source Integrity

Determinism is maintained end-to-end, from belief revision semantics through source integrity verification (determinism-spans-revision-semantics-through-source-integrity). Revision operations produce deterministic, traceable state transitions — structured before/after diffs with controlled irreversibility. The source verification that triggers those revisions is itself deterministic and architecturally grounded: fail-safe path resolution, exact SHA-256 comparison, and clean layer boundaries.

The significance of this span is that no step in the revision-triggered-by-source-change pathway introduces non-determinism. A given source change will always produce the same belief revision, and that revision will always produce the same state transition. This property is foundational to the system's governance guarantees, supporting the broader claim that governance spans revision through source integrity ([rich-governance-determinism-spans-revision-through-source](governance.md#rich-governance-determinism-spans-revision-through-source)).

## Exception Safety Across TMS and Source Lifecycle

The system handles failures safely across both the TMS core and the source integrity pipeline (exception-safety-spans-tms-and-source-lifecycle). The TMS core resolves exceptional conditions — contradictions via backtracking, challenges via deterministic propagation. The source integrity pipeline handles its own failure modes — missing files, changed content, deleted sources. No exception in either domain can produce inconsistent or undefined state.

This span is particularly important because the TMS and source pipeline interact: source changes trigger belief revisions, and if either side could fail into an undefined state, the interaction boundary would become a vulnerability. By ensuring exception safety on both sides independently, the system guarantees that the composed pathway is also exception-safe — supporting the higher-level claims about both governance exception safety ([rich-governance-is-exception-safe](governance.md#rich-governance-is-exception-safe)) and source-to-TMS integrity ([source-to-tms-integrity-is-deterministic-and-exception-safe](source.md#source-to-tms-integrity-is-deterministic-and-exception-safe)).

## Cross-Cutting Theme

The span beliefs share a common pattern: they take two independently verified properties and assert that their *composition* holds. This is non-trivial — subsystem guarantees do not automatically compose across boundaries. Each span belief depends on its constituent properties being established first, then makes the additional claim that no gap exists at the interface between them. This layered verification structure — prove the parts, then prove the whole — is characteristic of the system's approach to architectural assurance.
