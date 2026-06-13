# complete

[Back to index](index.md)

The concept of completeness pervades the ftl-reasons architecture, appearing at every layer from low-level evaluation primitives through high-level dialectical operations. Rather than a single property, completeness is a family of guarantees: that every negation form is covered, every truth change propagates fully, every contradiction resolves traceably, and every revision reverses cleanly. These guarantees reinforce each other — propagation completeness enables cascade completeness, which enables revision completeness, which enables architectural completeness.


## Architectural Completeness

The core architectural claim is that the reasoning-and-revision system is both deterministic in its state trajectories and lifecycle-complete in its monitoring coverage — no belief can escape either guarantee (complete-architecture-is-deterministic-and-lifecycle-complete). This dual property means every belief is predictably computed and fully tracked throughout its existence.

This deterministic lifecycle-complete architecture achieves full portability across storage backends, operating identically on both SQLite and PostgreSQL (complete-architecture-is-backend-portable). The portability claim rests on uniform safety enforcement across all layers and backends, combined with the architecture's predictable state trajectories.

An earlier, stronger claim — that this architecture achieves verified production correctness across all operation types — has been retracted (complete-architecture-achieves-verified-production-correctness, OUT), as has the claim that completeness and invariant preservation flow from shared minimal foundations (complete-architecture-preserves-invariants-minimally, OUT). These retractions reflect refinements in understanding rather than fundamental architectural flaws.


## Complete Negative Semantics

The system achieves complete semantics for all forms of negation through two complementary mechanisms (absence-and-outlist-form-complete-negative-semantics). Structural absence produces emergent premise behavior and asymmetric fail modes, while explicit outlist entries provide conjunctive defeat with defined absent-node handling and persistence. Together, these cover every mechanism by which beliefs can be negated or defeated.

These negative semantics form a complete belief modification lifecycle: all defeat mechanisms reverse automatically through BFS propagation cascades, and surgical restoration hints target only cascade victims with surviving premises (negative-semantics-are-complete-reversible-and-recoverable). Every belief retraction can be undone with guided recovery.

Uniform handling of semantic edge cases — vacuous premises, asymmetric absence, empty antecedents — reinforces this completeness (edge-case-uniformity-reinforces-complete-negative-semantics). Every edge case within the negation lifecycle is handled by the same minimal evaluation rules that ensure reversibility, rather than requiring special-case logic.


## Propagation and Dependency Completeness

Incremental truth propagation reaches every node whose truth value should change — including nodes depending via outlist entries — without requiring periodic full recomputation (incremental-propagation-is-fully-complete). This works because safe terminating BFS traversal operates over a dependents index that tracks both antecedent and outlist relationships.

The dependents index itself is complete for all reference types (dependency-tracking-is-complete-for-all-reference-types), maintained incrementally so that every network mutation keeps it current. This completeness has a concrete downstream effect: deduplication survivor selection accurately reflects the full dependency graph, preserving the structurally most-connected node in each cluster (dedup-reflects-complete-dependency-graph).

The claim that both forward propagation and backward retraction achieve complete graph traversal with guaranteed termination has been retracted (graph-traversal-is-complete-and-terminating-in-both-directions, OUT), suggesting that bidirectional completeness remains an area of ongoing refinement.


## Contradiction and Revision Completeness

Contradiction management is complete and traceable along two axes: the revision pipeline reliably resolves contradictions through outlist defeat and dependency-directed backtracking with guaranteed termination, while nogood resolution maintains a consistent referenceable history of all detected contradictions (contradiction-management-is-complete-and-traceable). This enables both automated resolution and post-hoc forensic analysis.

Contradiction resolution simultaneously achieves minimal disruption and complete effect propagation (contradiction-resolution-achieves-minimal-impact-complete-cascades). Least-entrenched culprit selection with surgical recovery hints minimizes blast radius, while transitive retraction cascades ensure no dependent node escapes the cascade.

Belief modification is complete in both directions: contradictions creating truth changes resolve through traceable dependency-directed backtracking, and defeats suppressing truth values reverse automatically through BFS propagation with surgical restoration hints (belief-modification-is-bidirectionally-complete-and-traceable). No truth change is either irresolvable or irreversible without full traceability.

The claim that retraction reporting reflects complete cascades has been retracted (retraction-reporting-reflects-complete-cascades, OUT), conditioned on the dependents index tracking all relationship types including outlists.


## Dialectical Completeness

The recursive challenge/defend dialectical system inherits fully-specified semantics from the outlist primitive: conjunction over multiple outlists, absent-means-OUT permissiveness, and persistence guarantees all apply to dialectical structures without additional rules (dialectics-inherit-complete-outlist-semantics).

Dialectical operations achieve both semantic grounding and operational completeness (grounded-dialectics-achieve-complete-bidirectional-assurance). Evaluation purity and uniform semantics provide the grounding; forward reliability of challenge/defend paired with backward recovery of defeat reversal provides the bidirectional completeness.

The stronger claim that dialectics complete the revision system — handling both automated revision and interactive dialectics through the same outlist primitive — has been retracted (dialectics-complete-the-revision-system, OUT), along with the claim that identity transformation is complete and reliable in both directions (identity-transformation-is-complete-and-reliable, OUT).


## Backend Completeness

Both storage backends provide atomic isolated operations with complete API coverage (dual-backend-atomicity-is-feature-complete). SQLite's context-managed mutations and PgApi's per-method transactions cover the same full set of operations.

However, the claim that PgApi is a complete SQL-native reimplementation of the in-memory Network has been retracted (pgapi-is-complete-sql-reimplementation, OUT), as has the claim of referentially complete multi-tenancy (pg-multi-tenancy-is-referentially-complete, OUT). The latter was conditioned on potential phantom node references from JSONB-stored antecedent arrays lacking foreign key constraints.


## Evaluation Purity as Completeness Enabler

A key architectural insight is that evaluation purity — uniform, deterministic, side-effect-free justification checking — is the computational property making the completeness-minimality unification possible (evaluation-purity-enables-complete-minimal-architecture). Purity simultaneously grounds context-agnosticism (the minimal primitive set handles all cases uniformly) and ensures that architectural completeness follows from minimality rather than despite it.


## Retracted System-Level Claims

Several ambitious system-level completeness claims have been retracted, reflecting the knowledge base's epistemic discipline:

- Complete system self-correction across two independent dimensions (complete-system-is-self-correcting, OUT)
- Complete deterministic traceable history of all state changes (complete-system-history-is-deterministic-and-traceable, OUT)
- Complete quality lifecycle spanning creation through egress (complete-quality-lifecycle-spans-creation-through-egress, OUT)
- Complete operational uniformity across all mutation sources (complete-operational-uniformity-across-all-sources, OUT)
- Production readiness of the fully unified system (complete-unified-system-is-production-ready, OUT)
- Convergent equilibria with complete propagation fidelity (convergent-equilibria-have-complete-propagation-fidelity, OUT)

These retractions typically cascade from specific technical issues — propagation gaps, unresolved bugs, or dependency tracking limitations — rather than from fundamental design flaws. The pattern illustrates how the belief network's retraction mechanism prevents overclaiming: high-level completeness claims depend on lower-level guarantees, and when any foundation weakens, the derived claims retract automatically.
