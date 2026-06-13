# system

[Back to index](index.md)

## Overview

The "system" beliefs describe ftl-reasons as a unified architecture — how its minimal primitives compose into a complete reasoning engine with properties spanning correctness, self-maintenance, boundary safety, and scalability. Two foundational claims remain actively held: that the TMS derives entirely from minimal uniform primitives (`system-semantics-are-minimal-and-complete`), and that a single reversible outlist mechanism underlies all non-monotonic features (`non-monotonic-system-is-single-reversible-primitive`). The majority of higher-level system claims have been retracted, reflecting that the ambitious composite guarantees they described depend on lower-level properties — particularly around propagation correctness and the dependents index — that are not yet fully assured.

## Semantic Minimality and the Outlist Primitive

The most fundamental active claim is that the entire TMS derives from a minimal set of uniform primitives (`system-semantics-are-minimal-and-complete`). Monotonic truth maintenance uses emergent truth rules — disjunction over conjunction and premise-from-absence — while all non-monotonic defeat mechanisms rest on a single primitive: the outlist. Challenges, kill-switches, supersession, and dialectical structures are all built from outlist entries, with no dedicated machinery for any specific defeat pattern (`non-monotonic-system-is-single-reversible-primitive`). This primitive is inherently reversible — retracting the defeating belief restores the defeated one — giving the system its characteristic ability to undo any reasoning step.

These two beliefs jointly support the (now retracted) claim that the system is minimal, sound, and scalable (`system-is-minimal-sound-and-scalable`), and that when combined with complete dialectical revision, the design is unified across all features (`system-is-unified-minimal-dialectical-and-scalable`). The retraction of these derived claims traces back to dependencies on multi-agent soundness and integrity properties that have unresolved conditions.

## Artifact Identification

System-generated artifacts maintain consistent, referenceable identification (`system-artifacts-maintain-consistent-identification`). Challenge nodes receive deterministic auto-generated IDs, and contradiction records (nogoods) are unconditionally recorded with stable identifiers. This means system-generated artifacts are as addressable as user-created beliefs — a property that supports traceability across mutations, self-correction artifacts, and the broader goal of making all system history consistently referenceable.

## Correctness and Crash Safety

A retracted composite claim held that the system achieves correctness at every level: deterministic conservative truth maintenance, a single reversible primitive for non-monotonic features, and data integrity spanning all architectural layers (`system-achieves-full-correctness`). Similarly, a claim that system operations never crash (`system-operations-never-crash`) was qualified by the condition that no dangling dependent references exist in the graph. Since the dependents reverse index has known maintenance gaps — outlist nodes are not tracked in it — both claims were retracted. The reasoning engine's determinism and reversibility remain well-supported, but the integrity layer they depend on has caveats.

## Self-Maintenance and Convergence

A substantial cluster of beliefs describes the system as a self-maintaining loop that autonomously converges to consistent states. The claimed convergence operates bidirectionally: import reconciliation converges through fixpoint iteration when beliefs are added, while retraction cascades terminate through BFS with stop-on-unchanged when beliefs are removed (`system-converges-from-addition-and-removal`). Together with deduplication, these paths were said to guarantee that no modification leaves the network in an oscillating state (`system-reaches-equilibrium-from-all-modification-paths`).

Active self-correction was claimed to complement this passive convergence: contradiction resolution handles inconsistencies at derivation time, while staleness detection catches degraded beliefs at maintenance time (`system-autonomously-converges-and-self-corrects`). The maintenance loop was further described as fully characterized — origin-agnostic, fully observable, and self-sustaining through minimality's fixed-point property (`system-is-fully-characterized-self-maintaining-loop`).

All of these convergence and self-maintenance beliefs are currently OUT. Their retraction cascades from dependencies on properties like complete self-correction and import reconciliation correctness that themselves depend on unresolved lower-level conditions.

## Boundary Safety and Evolution Tolerance

The system's external boundaries were described as simultaneously enforcing strict validation and tolerating format evolution (`system-boundaries-are-validating-and-evolution-tolerant`). On the validation side: typed exceptions, referential integrity checks, and LLM hallucination filtering reject malformed input. On the evolution side: dual-format derive parsers with automatic fallback, forward-compatible metadata handling in belief import, and SQLite schema migration via try/except allow the system to handle older or newer formats gracefully (`system-tolerates-evolution-at-all-boundaries`).

Reference validation was characterized as defense-in-depth — import normalization drops unknown references, nogoods skip missing nodes, and hallucination filtering discards phantom IDs (`system-boundaries-are-evolution-tolerant-and-reference-safe`). A further claim added resource governance to this picture, with bidirectional token budgets and access-tag gating (`system-boundary-enforcement-spans-validation-resilience-and-resources`).

These boundary beliefs are all retracted, primarily because their composite claims aggregate properties whose individual foundations have gaps — particularly around comprehensive input validation and evolution tolerance being proven across all boundaries rather than just the ones examined.

## Operational Assurance

Several beliefs attempted to characterize the system's operational assurance as spanning multiple independent dimensions. The tripartite model described the system as externally controlled (bounded interfaces with defensive ingestion), internally self-correcting (contradiction resolution and staleness detection), and query-resilient (graceful degradation across access paths) (`system-achieves-tripartite-operational-assurance`). This was extended to claim universality across all architectural layers, storage backends, and adverse graph conditions (`system-assurance-is-universal-and-multidimensional`).

Reliability was said to span both internal dimensions (read paths via staleness detection, write paths via crash-free propagation) and external dimensions (bidirectionally bounded interfaces) (`system-reliability-spans-internal-and-external`). These layered assurance claims are all OUT, reflecting that the individual reliability and control properties they compose have unresolved dependencies.

## Resource Efficiency

The system was characterized as resource-efficient across all lifecycle phases (`system-resource-footprint-is-minimal-at-all-phases`): zero external dependencies eliminate installation overhead, lazy module imports defer heavy computation, and O(1) per-line budget tracking with chars/4 token estimation minimizes runtime overhead (`system-efficiency-spans-packaging-and-runtime`). These claims supported a broader narrative of resource-efficient self-sustainability — that the system's efficiency prevents exhaustion of its own maintenance mechanisms (`system-is-resource-efficient-self-sustaining-and-auditable`). All are retracted due to upstream dependencies.

## Output Governance

System output was described as comprehensively governed along multiple dimensions: normalized with uniform fail-safe schemas, authorized through access-tag gating with transitive inheritance, and resource-constrained through bidirectional token budgets (`system-output-is-comprehensively-governed`). The two primary output mechanisms — compact belief summaries and staleness reports — were claimed to meet production standards including CI-pipeline readiness with deterministic sorted output and nonzero exit codes (`system-output-is-complete-bounded-and-ci-ready`). Both claims are OUT.

## Universal and Permanent Guarantees

The most ambitious retracted claims assert that the system's guarantees are simultaneously universal (extending to all belief types including external ones), permanent (maintained indefinitely without degradation), and verifiable (auditable without trusting internal state) (`system-guarantees-are-universal-permanent-and-verifiable`). These depend on chains reaching back to self-sustainability, invariant preservation, and origin-agnostic observability — properties that themselves rest on the self-maintaining loop and comprehensive invariant coverage claims.

The claim that all system properties emerge from a single unified design rather than independent engineering (`system-properties-emerge-from-unified-design`) captures the architectural aspiration: integrity and scalability as complementary consequences of unified internal/external integrity, extensibility and robustness as joint products of minimality. While the foundational minimality claims remain IN, the emergent-properties superstructure is retracted.

## Unified Architecture

The capstone belief described the system as a closed self-maintaining belief architecture where end-to-end integrity prevents corruption and revision completeness ensures any valid configuration is reachable — but only when all known defects and fragilities are resolved (`unified-system-is-a-closed-self-maintaining-architecture`). This conditional qualification is telling: the architectural vision is coherent, but its realization depends on resolving the dependents index gap and ensuring propagation correctness across all paths.

The pattern across these system-level beliefs is consistent. The two IN beliefs establish that the design is genuinely minimal and built on a single reversible primitive. The retracted beliefs represent derived conclusions about what that minimal design *should* guarantee — conclusions that remain architecturally sound but depend on implementation-level properties that are not yet fully verified.
