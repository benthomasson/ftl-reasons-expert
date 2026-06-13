# self

[Back to index](index.md)

The ftl-reasons system exhibits a cluster of "self-*" properties — self-containment, self-correction, self-sustainability, and self-documentation — that collectively describe a system designed to maintain its own consistency without external intervention. The foundational claim is architectural: the project is both externally self-contained and internally well-structured (`architecture-is-self-contained-and-safely-layered`). Most derived claims about what this architecture enables have been retracted, reflecting revised confidence in the scope of these guarantees.

## Self-Contained Architecture

The one currently-held belief in this cluster establishes that the project achieves self-containment at two levels: zero runtime dependencies externally, and clean three-layer boundaries internally (`architecture-is-self-contained-and-safely-layered`). This architectural foundation was expected to support two key downstream properties — that self-correction requires no external dependencies (`self-correction-requires-no-external-dependencies`) and that trust boundaries are architecturally enforced (`[trust-boundary-is-architecturally-enforced](other.md#trust-boundary-is-architecturally-enforced)`). Both of those derived beliefs have since been retracted.

## Self-Correction

The most extensively developed theme concerns the system's self-correction capabilities. The core mechanism operates along two temporal axes: *creation-time* contradiction resolution through dependency-directed backtracking, and *maintenance-time* staleness detection through source hash comparison (`self-correction-spans-creation-and-maintenance`). Together these were characterized as making the system self-correcting across its complete belief lifecycle (`self-correction-is-exhaustive-across-lifecycle`).

Several belief chains elaborated on the quality dimensions of self-correction. One line established that correction is concretely grounded in source-level integrity (fail-safe path resolution, SHA-256 hashing), fully self-documenting through artifact production, and convergent through accurate topology tracking (`self-correction-is-grounded-documented-and-convergent`). Another established resource sustainability — that accurate bidirectional token budgets prevent the correction loop from exhausting its resource envelope (`self-correction-is-resource-sustainable`). A third addressed evolution tolerance, claiming that parser fallbacks and forward-compatible import parsing ensure correction mechanisms survive format changes (`self-correction-is-evolution-tolerant-and-sustainable`).

These dimensions were synthesized into a comprehensive claim that self-correction achieves all quality properties simultaneously without trading any off against another (`self-correction-is-complete-across-all-quality-dimensions`). That belief, along with its entire support chain, is currently OUT.

### Resilience to LLM Unavailability

A notable property of the self-correction design is its relationship to LLM availability. Because the core mechanisms — backtracking and hash comparison — operate on stdlib alone, LLM unavailability degrades knowledge *expansion* but never compromises correction *integrity* (`self-correction-is-resilient-to-llm-unavailability`). This separation between LLM-dependent growth and LLM-independent maintenance was a deliberate architectural choice.

### Backend Independence

The self-correction pipeline was also characterized as backend-independent: contradiction resolution, staleness detection, and belief currency management were claimed to operate identically across all storage backends (`backend-independent-self-correction`). This would make self-maintenance a deployment-independent guarantee rather than a backend-specific capability. This belief is OUT, dependent on retracted claims about both the self-correction system and storage layer agnosticism.

## Self-Sustainability and Minimality

The deepest theoretical claim in this cluster concerns minimality as a fixed point. The argument runs: minimality generates the closed forward/backward maintenance loop *and* the self-correction mechanisms that maintain that loop, so the generative principle sustains itself through its own consequences (`minimality-is-self-sustaining`). This creates a meta-level consistency loop where the minimal primitives that preserve system invariants are themselves invariants of the system (`invariant-preservation-is-self-sustaining`).

This self-sustainability property was further claimed to be reinforced by resource efficiency — zero external dependencies eliminate supply-chain risk, lazy loading reduces overhead, and O(1) budget tracking bounds computational cost (`self-sustainability-is-reinforced-by-resource-efficiency`). The combined result was characterized as a system that is simultaneously resource-efficient, self-sustaining, and auditable.

An important qualification: the self-sustaining invariant preservation was claimed to be *independently verifiable* rather than requiring blind trust, through the same maintenance loop observability that enables trustworthiness verification (`self-sustaining-invariants-are-independently-verifiable`). This verifiability claim, like the rest of the chain, is currently OUT.

## Self-Documenting Audit Trail

Every self-correction was claimed to produce consistently identifiable artifacts — deterministic challenge IDs and monotonic collision-free nogood IDs — enabling a complete referenceable correction history (`self-correction-produces-referenceable-artifacts`). This documentation was characterized as durable across persistence boundaries and format evolution (`self-correction-history-is-durably-documented`), forming an indefinitely referenceable audit trail (`self-correction-audit-trail-is-permanent-and-comprehensive`).

The audit trail feeds into broader claims about indefinite operation: if self-correction sustains the lifecycle indefinitely and every correction is fully auditable, then auditability scales with time rather than decaying (`indefinite-self-correction-is-fully-auditable`). The fully characterized self-maintaining loop provides complete operational auditability across all belief origins and correction types (`self-maintenance-is-fully-auditable`), conditional on propagation soundness guaranteeing faithful cascade recording.

## Trust Boundaries

Trust boundaries were characterized as both structurally enforced and dynamically self-maintaining: zero external dependencies and defensive ingestion provide static guarantees, while autonomous convergence ensures every modification reaches a deterministic stable state within trust boundaries (`trust-boundaries-are-self-maintaining`). This would mean the system's trust guarantees require no external enforcement mechanism.

## Current Status

Of the beliefs in this cluster, only the foundational architectural self-containment claim remains IN. The extensive derived chains — spanning self-correction completeness, self-sustainability through minimality, self-documenting audit trails, and self-maintaining trust boundaries — are all OUT. This pattern suggests that while the architectural foundation holds, the elaborate guarantees built upon it were either over-specified or dependent on other retracted claims about storage backends, lifecycle soundness, or topology accuracy. The retraction of upstream beliefs like `[complete-system-is-self-correcting](complete.md#complete-system-is-self-correcting)` and `[minimality-sustains-closed-loop-maintenance](other.md#minimality-sustains-closed-loop-maintenance)` cascaded through these dependency chains, pulling out most of the "self-*" cluster despite the architectural base remaining intact.
