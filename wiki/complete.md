# complete

[Back to index](index.md)

The concept of completeness in ftl-reasons spans multiple dimensions of the system's design — from the semantics of negation to the architecture of truth propagation, contradiction resolution, and dialectical reasoning. Rather than a single property, completeness is an emergent guarantee that arises when each subsystem covers its full domain without gaps, ensuring no belief escapes evaluation, no truth change goes unpropagated, and no defeat is irreversible.

## Negative Semantics

The system achieves complete coverage of all forms by which beliefs can be negated or defeated. Two complementary mechanisms handle the full negation space: structural absence produces emergent premise behavior with asymmetric fail modes, while explicit outlist entries provide conjunctive defeat with defined absent-node handling and persistence (absence-and-outlist-form-complete-negative-semantics). Together these mechanisms form a complete belief modification lifecycle — every form of defeat reverses automatically through BFS propagation cascades, and surgical restoration hints target only cascade victims with surviving premises, so every retraction can be undone with guided recovery (negative-semantics-are-complete-reversible-and-recoverable).

This completeness is reinforced by the system's uniform handling of semantic edge cases. Vacuous premises, asymmetric absence, and empty antecedents are all governed by the same minimal evaluation rules that ensure reversibility (edge-case-uniformity-reinforces-complete-negative-semantics). The uniformity is not incidental — it follows directly from the minimality of the evaluation primitive set, which handles all cases without case-specific logic.

## Belief Modification and Contradiction Resolution

Belief modification is complete in both directions. When contradictions arise, they are resolved through traceable dependency-directed backtracking with consistent nogood IDs. When defeats suppress truth values, they reverse automatically through BFS propagation with surgical restoration hints. No truth change is either irresolvable or irreversible without full traceability (belief-modification-is-bidirectionally-complete-and-traceable).

The contradiction management pipeline itself provides complete coverage: the revision pipeline reliably resolves contradictions through outlist defeat and backtracking with guaranteed termination, while nogood resolution maintains a consistent referenceable history of all detected contradictions — enabling both automated resolution and post-hoc forensic analysis (contradiction-management-is-complete-and-traceable). Notably, contradiction resolution simultaneously achieves two properties that might seem to be in tension: minimal disruption through least-entrenched culprit selection, and complete effect propagation through transitive retraction cascades with guaranteed termination. The system minimizes blast radius while ensuring no node escapes the cascade (contradiction-resolution-achieves-minimal-impact-complete-cascades).

## Dependency Tracking and Propagation

Complete dependency tracking is foundational to several other completeness guarantees. The dependents index fully tracks all relationship types — both antecedent references and outlist references — enabling complete incremental propagation for every truth value change without requiring periodic full recomputation as a fallback (dependency-tracking-is-complete-for-all-reference-types). This matters because incremental truth propagation must reach every node whose truth value should change, including nodes that depend via outlist entries. Safe terminating BFS traversal over this complete index achieves exactly that (incremental-propagation-is-fully-complete).

Dependency completeness also enables accurate deduplication: survivor selection reflects the complete dependency graph by preserving the node with the most dependents in each cluster, counting both antecedent-based and outlist-based edges (dedup-reflects-complete-dependency-graph).

## Architectural Completeness

At the architectural level, completeness manifests as the conjunction of deterministic state trajectories and gapless lifecycle monitoring — no belief can escape either guarantee, creating a system where every belief is both predictably computed and fully tracked (complete-architecture-is-deterministic-and-lifecycle-complete). This deterministic, lifecycle-complete architecture is fully portable across storage backends, operating identically on both SQLite and PostgreSQL (complete-architecture-is-backend-portable). Both backends provide atomic isolated operations with complete API coverage, with SQLite's context-managed mutations and PgApi's per-method transactions covering the same full set of operations (dual-backend-atomicity-is-feature-complete).

A deeper architectural insight is that evaluation purity — uniform, deterministic, side-effect-free justification validity checking — is the concrete computational property that makes completeness and minimality compatible rather than opposing. Purity simultaneously grounds context-agnosticism and ensures that architectural completeness follows from rather than despite the minimal primitive set (evaluation-purity-enables-complete-minimal-architecture).

## Dialectical Completeness

The dialectical subsystem inherits its completeness guarantees from the outlist primitive rather than defining its own. The recursive challenge/defend structure inherits fully-specified semantics: conjunction over multiple outlists, absent-means-OUT permissiveness, and persistence guarantees all apply to dialectical structures without additional rules (dialectics-inherit-complete-outlist-semantics). Dialectical operations achieve both semantic grounding through evaluation purity and operational completeness through forward reliability of challenge/defend with backward recovery of defeat reversal (grounded-dialectics-achieve-complete-bidirectional-assurance).

## Relationships Between Completeness Properties

The various completeness properties are not independent — they form a dependency structure where lower-level guarantees compose into higher-level ones. Complete outlist semantics ([outlist-semantics-are-fully-specified](other.md#outlist-semantics-are-fully-specified)) is a particularly load-bearing dependency: it directly supports the completeness of negative semantics, dependency tracking, incremental propagation, and dialectical inheritance. Similarly, the completeness of contradiction management feeds into both bidirectional modification completeness and the broader architecture's determinism and traceability guarantees. The overall picture is one of layered completeness, where each subsystem's coverage guarantee contributes to the system's ability to handle every belief state without gaps or unrecoverable failures.
