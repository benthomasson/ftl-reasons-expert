# topology

[Back to index](index.md)

The topology of a Truth Maintenance System describes how state changes propagate through the dependency graph — not just to direct dependents, but to every transitively reachable node, including those connected through outlist (defeat) relationships. In ftl-reasons, topology completeness is a foundational invariant: no truth change, metadata update, or structural modification leaves any part of the graph in an inconsistent or unreached state.

## Topology-Complete Propagation

The core guarantee is that truth propagation reaches all transitively dependent nodes, including those connected through outlist entries, not just antecedent references (propagation-is-topology-complete-and-inconsistency-safe). This distinction matters because outlist connections represent non-monotonic defeat relationships — a node whose outlist member changes truth value may itself need to flip, and that flip must cascade further.

Propagation is also safe under graph inconsistency: if the network contains dangling references — nodes deleted without full reference cleanup — the system skips them with structured warnings rather than crashing. This means cascading remains correct even in networks with imperfect structural integrity.

These two properties — completeness and inconsistency safety — combine to ensure that all forms of truth change achieve what the system calls "topology-complete multi-dimensional consistency" (all-truth-changes-are-topology-complete-and-robust). Truth values, the dependents index, and access tags all cascade together, so no dimension falls out of sync during propagation.

## State Transitions and Traceability

Every belief state transition in the system satisfies four simultaneous guarantees: it is topology-complete, deterministic with traceable history through structured diffs, robust under graph inconsistency, and never produces an untraced or incomplete state change (all-state-transitions-are-topology-complete-and-traceable).

These transitions are further exception-safe and recoverable (topology-complete-transitions-are-exception-safe). When contradictions or challenges arise, the system handles them without corruption and provides guided restoration hints for cascade victims — nodes that were retracted as a side effect of resolving a contradiction elsewhere in the graph. This means that even adversarial graph conditions cannot produce a topology-incomplete or corrupted state.

## Bidirectional Modification

Topology completeness applies in both directions of belief modification. Forward modification — contradiction resolution through traceable backtracking — and backward modification — defeat reversal with guided recovery — both propagate through the complete dependency graph (bidirectional-modification-achieves-topology-completeness).

Defeat reversal deserves particular attention: when a defeated belief is restored, the recovery propagates through topology-complete inconsistency-safe cascades, reaching all transitively affected nodes including outlist-connected ones and handling dangling references gracefully (defeat-reversal-is-topology-complete-with-guided-recovery). Recovery hints target only cascade victims with surviving premises, avoiding unnecessary restoration attempts.

All truth effects — incremental changes and outlist-based defeat reversals alike — maintain full traceability through transitive cascades with audit logging and consistent artifact identification (all-truth-effects-are-topology-complete-and-traceable).

## Topology Preservation in Structural Operations

Structural operations on the belief graph — not just truth changes but modifications to the graph itself — must also preserve topology. The system's two belief replacement mechanisms illustrate this.

Deduplication preserves network topology by rewriting both antecedent and outlist references to the surviving node, selecting structurally-optimal survivors based on dependent count with lexicographic tiebreak, and supporting human oversight through user-editable KEEP/RETRACT plans (dedup-is-topology-preserving-and-auditable). Survivor selection is reliable because the dependents index accurately reflects the justification graph (dedup-survivor-selection-is-topology-reliable).

Supersession operates through reversible outlist semantics with gated view exclusion, while deduplication rewires all justification references to the most-connected survivor. Both mechanisms ensure the dependency graph remains structurally sound and consumers see a clean, non-redundant belief set (belief-replacement-is-topology-safe-and-view-consistent).

## Governance and Rich State

Topology completeness extends beyond binary truth values into the system's richer governance framework. Metadata — retraction flags, stale reasons, access tags, and supersession markers — propagates topology-completely through the same cascades that carry truth changes (metadata-governed-modifications-are-bidirectional-and-topology-complete). There is no governance gap between forward and backward modifications; every direction of change carries complete metadata through the entire dependency graph.

Every topology-complete transition within this framework produces metadata-enriched traceable state (topology-complete-governance-produces-rich-traceable-state). The governance system operates at two levels of exception safety — both the propagation mechanics and the governance framework itself handle exceptions without corruption (topology-complete-transitions-within-rich-governance).

## Design Implications

The topology invariants form a layered guarantee stack. At the base, propagation is complete and inconsistency-safe. Above that, state transitions add traceability and determinism. Exception safety and recoverability provide resilience. Bidirectional modification ensures both contradiction resolution and defeat reversal respect the same completeness properties. Rich governance extends these guarantees beyond truth values to all managed metadata. Each layer depends on the one below it, and together they ensure that the belief network can never reach a state where some part of the graph has been left behind by a change that should have reached it.
