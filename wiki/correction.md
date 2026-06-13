# correction

[Back to index](index.md)

Correction in the ftl-reasons belief network concerns how changes to belief status propagate through the dependency graph and whether those propagations converge reliably.

## Convergence and Topology

A central conjecture held that knowledge growth converges to stable equilibria where every correction operates on accurate topology — meaning dependency completeness ensures corrections propagate through the true graph structure rather than an approximation (`knowledge-equilibria-are-correction-convergent-and-topology-accurate`). This belief synthesized two claims: that all corrections converge on accurate topology (`all-corrections-converge-on-accurate-topology`) and that knowledge growth reaches negation-transparent equilibria (`knowledge-growth-reaches-transparent-equilibria`).

This composite belief is currently **OUT**, indicating that one or both of its supporting claims have been retracted. The retraction is notable because it qualifies an optimistic view of correction propagation — in practice, the system's handling of corrections may not always operate on complete or accurate dependency information. This aligns with the known limitation that outlist nodes are not tracked in the dependents index, which can cause corrections (particularly retractions) to fail to propagate through all affected paths.

## Practical Implications

The retracted status of this belief serves as a caution: while the TMS aims for faithful propagation of corrections through the dependency graph, the current implementation does not guarantee that every correction reaches all beliefs it should affect. GATE beliefs, in particular, may require manual re-evaluation after upstream retractions rather than being automatically corrected by the propagation mechanism.
