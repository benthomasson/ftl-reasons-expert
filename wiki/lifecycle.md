# lifecycle

[Back to index](index.md)

The belief lifecycle in ftl-reasons encompasses every state transition a node can undergo — from creation through staleness detection, propagation, contradiction resolution, and retraction — with the guarantee that no operation ignores or corrupts lifecycle state. This gapless coverage is not incidental but emerges from the interaction of metadata governance, architectural layering, and deterministic propagation mechanics.

## Gapless Lifecycle Management

The system manages belief lifecycle without gaps across all operation types (lifecycle-management-is-gapless). This property rests on two pillars: staleness checking detects all forms of source drift, and propagation respects node lifecycle states. Both read-only inspection and mutation-driven propagation honor lifecycle consistently — staleness checking skips OUT nodes without mutating state, while propagation skips retracted nodes and preserves trigger identity (lifecycle-awareness-spans-checking-and-propagation). The result is that lifecycle state is respected regardless of whether the operation is a read or a write.

This gapless property is further reinforced from two directions. Deterministic reasoning ensures predictable state trajectories with full monitoring, while architectural safety provides structural foundations through clean layer boundaries and atomic mutations (lifecycle-is-deterministic-and-architecturally-grounded). The architectural contribution is concrete: isolated mutations prevent cross-layer leakage, so lifecycle operations execute atomically (architecture-sustains-gapless-lifecycle).

## Metadata as Lifecycle Governor

A distinctive feature of the lifecycle model is that it transcends the binary IN/OUT truth values native to the TMS data model. Node metadata enables governance capabilities that go beyond this binary distinction, providing structured lifecycle state including retraction flags, stale reasons, access tags, and supersession markers (metadata-enables-lifecycle-governance-beyond-binary-truth). Staleness information, for instance, has no dedicated truth state in the TMS but is preserved and surfaced in compact output through metadata.

This metadata is not passive annotation — it actively governs system behavior across the complete operational surface (metadata-governs-lifecycle-across-read-and-write-paths). During truth propagation, retracted nodes are skipped based on metadata flags. During staleness checking, OUT nodes are excluded. The same metadata fields drive both mutation behavior (sticky retraction surviving recompute) and inspection behavior (compact surfacing stale reasons). This dual role makes metadata simultaneously the universal extension mechanism for network state and the active governor of propagation behavior (metadata-provides-extensible-lifecycle-governance).

## Propagation and Node Lifecycle

Truth propagation is lifecycle-aware by design. Retracted nodes are skipped during BFS traversal, and the trigger node itself is never recomputed — callers must set its truth value before invoking propagation (propagation-respects-node-lifecycle). This separation of concerns prevents propagation from overriding explicit lifecycle transitions while ensuring cascades terminate correctly.

## Contradiction Resolution

When contradictions arise, the resolution pipeline respects node lifecycle throughout. Backtracking identifies the least-entrenched culprit premise deterministically, then retraction cascades through BFS propagation that skips already-retracted nodes and stops on unchanged truth values (contradiction-resolution-is-lifecycle-safe). This ensures resolution terminates without disturbing lifecycle-inert nodes — nodes that have already been retracted or whose truth values would not change are left undisturbed.

## Defeat Reversals as Lifecycle Events

All outlist-based defeat mechanisms — challenge, kill-switch, and supersession — are first-class lifecycle events rather than bare truth flips. Every reversal produces not just a truth-value change but a fully governed lifecycle transition with metadata-tracked state (defeat-reversals-are-lifecycle-governed). This governance holds across all storage backends; the same lifecycle state transitions and safety guarantees apply whether backed by SQLite or PostgreSQL (defeat-reversals-are-lifecycle-governed-across-all-backends).

## Source Integrity and Lifecycle Governance

Lifecycle governance is concretely grounded in source integrity. The source pipeline — convention-based resolution, collision-resistant SHA-256 hashing, and comprehensive staleness detection — populates and verifies the metadata state that enables lifecycle governance, closing the loop between abstract lifecycle management and concrete source verification (lifecycle-governance-is-metadata-enabled-and-source-grounded).

This grounding is itself deterministic and architecturally sound: lifecycle decisions about staleness and belief currency rest on collision-resistant hashing within clean three-layer boundaries (lifecycle-governance-has-deterministic-source-integrity). The governance layer is also exception-safe, protected by the same exception handling that safeguards contradiction resolution and dialectical transformation (lifecycle-governance-is-exception-safe-and-source-grounded). Together, these properties achieve gap-free source coverage — every source file is verified, every lifecycle decision is grounded in verifiable source state, and no silent source gap undermines governance decisions (lifecycle-governance-achieves-gap-free-source-coverage).

## Automated Sync and External Lifecycle

Automated repeated sync safely reconciles external beliefs with complete lifecycle coverage. Idempotent, cascade-preserving sync combined with conservative staleness gating ensures no lifecycle gap between sync runs (automated-sync-achieves-full-lifecycle-coverage). This extends the internal lifecycle guarantees to the boundary between the system and its external sources, maintaining the same gapless property across synchronization cycles.
