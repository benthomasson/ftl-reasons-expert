# lifecycle

[Back to index](index.md)

The belief lifecycle in ftl-reasons encompasses every phase a node passes through — from creation and justification, through staleness detection and truth propagation, to retraction and potential restoration. The system is designed so that no operation ignores or corrupts lifecycle state, achieving what the belief network characterizes as "gapless" lifecycle management (`lifecycle-management-is-gapless`).

## Gapless Lifecycle Management

The core lifecycle guarantee is that every operation type — read or write — consistently honors the lifecycle state of every node. Staleness checking detects all forms of source drift without mutating state, while truth propagation skips retracted nodes and preserves trigger identity (`lifecycle-awareness-spans-checking-and-propagation`). This consistency across both inspection and mutation paths means a single set of lifecycle rules governs the system's complete operational surface (`metadata-governs-lifecycle-across-read-and-write-paths`).

Gapless management is reinforced from two directions. Deterministic reasoning ensures predictable state trajectories with full monitoring, while architectural safety provides structural enforcement through clean layer boundaries and atomic mutations (`lifecycle-is-deterministic-and-architecturally-grounded`). These architectural guarantees — isolated mutations and clean data-model/TMS/persistence boundaries — ensure that lifecycle operations execute atomically without cross-layer leakage (`architecture-sustains-gapless-lifecycle`).

A previously held belief extended this to a third reinforcement: verification that the underlying architecture itself contains no hidden fragilities (`lifecycle-is-deterministic-grounded-and-structurally-sound`, OUT). That belief was retracted, though the two remaining pillars — deterministic reasoning and architectural grounding — continue to hold.

## Metadata-Enabled Governance

The TMS data model supports only binary truth values (IN/OUT), but node metadata extends lifecycle governance well beyond this binary. Extensible metadata fields — retraction flags, stale reasons, access tags, supersession markers — provide structured lifecycle state that actively governs system behavior (`metadata-enables-lifecycle-governance-beyond-binary-truth`). Staleness information, for example, is preserved and surfaced in compact output despite having no dedicated truth state in the data model.

Metadata serves a dual role: it is both the universal extension mechanism for network state and the active governor of truth propagation behavior (`metadata-provides-extensible-lifecycle-governance`). Retracted nodes are skipped during propagation, stale nodes are skipped during staleness checking, and both behaviors are driven by the same metadata fields. This makes metadata the single mechanism controlling the system's complete operational surface.

## Source Integrity and Lifecycle Governance

Lifecycle governance is concretely grounded in source verification. The source pipeline — convention-based file resolution, collision-resistant SHA-256 hashing, and comprehensive staleness detection — populates and verifies the metadata state that enables governance decisions (`lifecycle-governance-is-metadata-enabled-and-source-grounded`). This closes the loop between abstract lifecycle management and concrete source verification.

The deterministic foundation is explicit: lifecycle decisions about staleness and belief currency rest on SHA-256 hashing within clean three-layer boundaries, ensuring source-grounding is itself structurally sound (`lifecycle-governance-has-deterministic-source-integrity`). Combined with exception-safe revision mechanics that protect contradiction resolution and dialectical transformation (`lifecycle-governance-is-exception-safe-and-source-grounded`), this achieves gap-free source coverage — every source file is verified, every lifecycle decision is grounded in verifiable source state (`lifecycle-governance-achieves-gap-free-source-coverage`).

## Propagation and Contradiction Resolution

Truth propagation respects node lifecycle at every step. Retracted nodes are skipped during BFS traversal, and the trigger node itself is never recomputed — callers must set its truth value before invoking propagation (`propagation-respects-node-lifecycle`).

Contradiction resolution is similarly lifecycle-safe. When contradictions are detected, backtracking identifies the least-entrenched culprit premise deterministically, and retraction cascades through BFS propagation that skips retracted nodes and halts on unchanged truth values (`contradiction-resolution-is-lifecycle-safe`). This ensures resolution terminates without disturbing lifecycle-inert nodes — nodes whose lifecycle state should not change are left untouched.

## Defeat Reversals as Lifecycle Events

All outlist-based defeat mechanisms — challenge, kill-switch, and supersession — operate within metadata-enabled lifecycle governance (`defeat-reversals-are-lifecycle-governed`). Every reversal produces not just a truth-value change but a fully governed lifecycle transition with metadata-tracked state. This makes reversals first-class lifecycle events rather than bare truth flips.

These guarantees hold across storage backends. The same lifecycle state transitions and safety guarantees apply regardless of whether the system is backed by SQLite or PostgreSQL (`defeat-reversals-are-lifecycle-governed-across-all-backends`).

## Automated Sync and External Beliefs

For beliefs originating from external sources, automated repeated sync safely reconciles external state with complete lifecycle coverage (`automated-sync-achieves-full-lifecycle-coverage`). Idempotent, cascade-preserving sync combined with conservative staleness gating ensures no lifecycle gap between sync runs — external beliefs receive the same lifecycle guarantees as internally derived ones.

## Quality Lifecycle (Retracted)

Several beliefs describing an LLM-driven quality lifecycle have been retracted. These described a pipeline spanning belief creation via `derive`, classification via `list-negative`, and validation via `review` — operating within resource-efficient bounds (`quality-lifecycle-is-complete-and-resource-efficient`, OUT) with fault tolerance at every phase (`quality-lifecycle-is-fault-tolerant-and-resource-efficient`, OUT). The pipeline was characterized as operating within self-maintaining trust boundaries requiring no external trust infrastructure (`quality-lifecycle-operates-within-self-maintaining-trust`, OUT).

A related belief noted that full code enforcement of quality constraints required the derive pipeline's minimum antecedent requirement to be enforced in code rather than prompt-only (`review-driven-quality-lifecycle-is-fully-code-enforced`, OUT). The retraction of these beliefs, along with the retraction of resource-sustainable lifecycle guarantees (`resource-sustainable-lifecycle-has-no-gaps`, OUT), suggests that the quality assurance dimension of lifecycle management is an area where the system's claims have been scaled back, while the core lifecycle machinery for truth propagation, metadata governance, and source integrity remains firmly held.
