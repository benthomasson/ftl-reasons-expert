# topology

[Back to index](index.md)

The belief network in ftl-reasons treats topology — the structure of dependency relationships between nodes — as a first-class concern. Topology completeness, preservation, and soundness appear throughout the system's guarantees about propagation, structural modification, and governance. The core principle is that every operation on the network must respect and maintain the graph structure: no dangling references, no unreached dependents, no silent structural corruption.

## Topology-Complete Propagation

The foundational guarantee is that truth propagation is topologically complete: when a node's truth value changes, the effect reaches every transitively dependent node, including those connected through outlist entries, not just antecedent references. Critically, this propagation is also safe under graph inconsistency — if a dependent reference points to a node that no longer exists, the system skips it with a structured warning rather than crashing (`propagation-is-topology-complete-and-inconsistency-safe`).

This property composes upward. All forms of truth change — both cascading propagation and justification addition — achieve topology-complete consistency across multiple dimensions: truth values, the dependents index, and access tags all cascade correctly even when the graph contains dangling references (`all-truth-changes-are-topology-complete-and-robust`). Truth effects including outlist-based defeat reversals propagate completely through outlist-connected paths while maintaining full traceability (`all-truth-effects-are-topology-complete-and-traceable`).

At the highest level, every belief state transition is simultaneously topology-complete, deterministic with traceable history, and robust under graph inconsistency — the system never produces a state change that is untraced, incomplete, or fragile (`all-state-transitions-are-topology-complete-and-traceable`).

## Topology Preservation in Structural Modifications

When the network's structure itself changes — through deduplication, belief replacement, or bulk operations — the system must preserve referential integrity. Deduplication provides the clearest example: it rewrites both antecedent and outlist references to point to the surviving node, selects the structurally-optimal survivor by choosing the node with the most dependents, and exposes a user-editable plan for human oversight (`dedup-is-topology-preserving-and-auditable`). The survivor selection is reliable precisely because the dependents index accurately reflects the justification graph (`dedup-survivor-selection-is-topology-reliable`).

Belief replacement more broadly achieves topology safety through two mechanisms. Supersession uses reversible outlist semantics with gated view exclusion, while deduplication rewires all justification references to the most-connected survivor. Both ensure the dependency graph remains structurally sound and consumers see a clean, non-redundant belief set (`belief-replacement-is-topology-safe-and-view-consistent`).

Several broader claims about bulk operations — that all bulk modifications preserve topology while converging deterministically (`bulk-operations-converge-and-preserve-topology`, OUT), and that all network modifications are simultaneously auditable and topology-preserving (`all-network-modifications-are-auditable-and-topology-preserving`, OUT) — have been retracted. The retraction of the convergence claim `topology-soundness-is-accurate-and-convergent` (OUT), which asserted that topology-modifying operations are both accurately measured and correctly maintained, reflects uncertainty about whether the combined guarantees hold universally. The individual mechanisms remain well-supported, but the synthesized claims linking convergence with topology preservation across all operation types do not currently hold.

## Bidirectional Completeness

Topology completeness applies in both directions of belief modification. Contradiction resolution traces backward through the dependency graph, while defeat reversal with guided recovery propagates forward — and both directions achieve topology-complete truth changes that are robust across all graph states (`bidirectional-modification-achieves-topology-completeness`).

Defeat reversal specifically combines automatic recovery guidance with topology-complete propagation: recovery reaches all transitively affected nodes including outlist-connected ones, handles dangling references gracefully, and provides restoration hints targeting only cascade victims with surviving premises (`defeat-reversal-is-topology-complete-with-guided-recovery`).

## Governance Through Topology

Topology completeness is not limited to truth-value propagation — it extends to the full governance framework. Bidirectional modifications carry metadata-governed lifecycle state (retraction flags, stale reasons, access tags) topology-completely within a deterministic lifecycle, ensuring no governance gap between forward and backward changes (`metadata-governed-modifications-are-bidirectional-and-topology-complete`).

Every topology-complete transition within this framework produces metadata-enriched traceable state, governing richer state than binary truth values including supersession metadata and access control (`topology-complete-governance-produces-rich-traceable-state`). These transitions are also exception-safe, handling contradictions and challenges without corruption while providing guided restoration hints for cascade victims (`topology-complete-transitions-are-exception-safe`). The exception safety is doubled — operating at both the propagation mechanics level and the governance framework level — within a richly-governed revision system (`topology-complete-transitions-within-rich-governance`).

## Convergence and Topology Soundness

Several beliefs attempted to synthesize topology preservation with deterministic convergence across all operations — asserting that every modification path reaches a stable state with topology fully preserved. These broader convergence claims are currently retracted. The claim that all belief replacements converge with topology preservation (`all-belief-replacements-converge-with-topology-preservation`, OUT) and that knowledge growth simultaneously preserves assurance while converging with topology preservation (`growth-converges-with-topology-and-assurance`, OUT) both depend on retracted intermediate beliefs about bulk operation convergence.

Similarly, claims linking topology accuracy with correction convergence have been retracted. The assertion that all corrections — both intentional dispute resolution and automated self-correction — converge on accurate topology (`all-corrections-converge-on-accurate-topology`, OUT) was retracted along with its dependencies on dispute resolution topology accuracy and self-correction topology accuracy.

The pattern across these retractions is consistent: the individual topology guarantees (completeness, preservation, robustness) remain well-supported at the mechanism level, but the synthesized claims that combine topology properties with universal convergence or universal accuracy across all operation types have not survived scrutiny. The system's topology handling is demonstrably sound for specific operations; what remains unresolved is whether these properties compose into the universal guarantees the retracted beliefs claimed.
