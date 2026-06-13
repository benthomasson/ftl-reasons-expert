# deterministic

[Back to index](index.md)

Determinism is a foundational property of the ftl-reasons Truth Maintenance System, permeating every layer from the core evaluation engine through dialectical operations, contradiction resolution, CLI output, and belief derivation. The system is designed so that identical inputs always produce identical outputs, every state transition is traceable, and no operation introduces opaque or unpredictable behavior.

## Core Engine Determinism

The TMS engine achieves deterministic, reversible non-monotonic reasoning through several reinforcing properties (reasoning-engine-is-deterministic-and-reversible). At its foundation, the core produces terminating truth maintenance via uniform pure evaluation, guaranteed convergence, and conservative asymmetric failure semantics for missing nodes (tms-core-is-deterministic-and-conservative). This means that justification evaluation follows the same rules everywhere, propagation always terminates, and the system fails safely rather than unpredictably when encountering absent nodes.

Determinism and lifecycle monitoring work in tandem: the system's belief-state trajectory is both fully determined and fully monitored (deterministic-reasoning-with-gapless-lifecycle). Any given set of premises produces exactly one truth-value assignment, while gapless lifecycle management ensures no belief escapes monitoring across any phase. The state at any point is predictable from its inputs and verifiable through its monitoring infrastructure.

## Dialectical Determinism

Dialectical operations — challenge and defend — inherit their determinism directly from the core engine rather than requiring independent proof. This works through semantic transparency: dialectical nodes are treated identically to ordinary beliefs by the deterministic evaluation engine, so no special-casing is needed (dialectics-are-deterministic-by-transparency). This transparency combines with full reliability — semantics-preserving behavior with crash safety through terminating propagation — to produce safe, predictable dialectical operations without dedicated dialectical machinery (dialectics-are-deterministic-and-reliable).

The grounding runs deeper still. Complete reversible negative semantics — structural absence producing emergent premise behavior, plus explicit outlist defeat with automatic reversal and guided recovery — form the foundation that enables deterministic reliable dialectics (negative-semantics-ground-deterministic-dialectics). Challenge and defend operations inherit determinism from evaluation purity applied to outlist primitives, and reliability from the inherent reversibility of outlist-based defeat.

This grounding chain extends transitively through uniform edge-case handling: uniformity in how vacuous premises, asymmetric absence, and empty antecedents are treated reinforces the negative semantics, which in turn ground dialectical correctness (uniform-semantics-transitively-ground-deterministic-dialectics).

## State Transitions and Contradiction Resolution

Every belief state change follows a deterministic evaluation path and produces a complete traceable history, whether initiated by intentional dialectical operations or automated contradiction resolution (all-state-transitions-are-deterministic-and-traceable). No state transition is opaque or unpredictable.

When contradictions are detected, resolution forms a deterministic pipeline: backtracking identifies the least-entrenched culprit premise, retraction triggers BFS propagation that terminates via a stop-on-unchanged condition, and the result is a new consistent state with minimal network disruption and guaranteed convergence (contradiction-triggers-deterministic-resolution).

## Lifecycle-Governed Modification

Bidirectional belief modification — contradiction resolution through traceable backtracking and defeat reversal with guided recovery — achieves topology completeness within a deterministic, architecturally-grounded lifecycle framework (bidirectional-modification-within-deterministic-lifecycle). This framework monitors every modification path from creation through maintenance, ensuring that both forward modification and backward recovery operate under the same deterministic guarantees.

## CLI and Output Determinism

The CLI achieves full scriptability through three deterministic properties: flat dictionary dispatch with no dynamic plugin resolution, binary exit codes (0 for success, 1 for error) with no ambiguous intermediate codes, and clean stream separation with diagnostics on stderr and results on stdout (cli-is-deterministic-and-stream-correct).

Staleness checking exemplifies output-level determinism: results are sorted by node ID, follow a uniform six-key schema across all result types, and return structured dictionaries for missing files rather than raising exceptions (check-stale-output-is-deterministic-and-structured). This produces output suitable for programmatic consumption and diffing.

## Derivation and Clustering Determinism

The derive subsystem maintains reproducibility through several mechanisms. Beliefs are sorted by ID before embedding, making cluster assignments reproducible given the same random seed (cluster-embed-order-is-deterministic). Cluster-based belief selection produces identical results for the same seed, returns exactly the requested budget count, and processes beliefs in sorted order — ensuring fully reproducible, precisely-sized belief subsets for derive prompt construction (cluster-selection-is-deterministic-and-budget-exact). Similarly, the sample mode with a fixed seed produces identical output across calls (sample-mode-is-deterministic).

Interestingly, one component deliberately breaks deterministic ordering: contradiction detection shuffles belief IDs randomly before batching so that repeated runs cover different pairwise combinations across batch boundaries (contradictions-shuffle-prevents-deterministic-batching). This controlled introduction of randomness serves the broader goal of thorough contradiction coverage — determinism within each run is preserved given the same shuffle seed, but cross-run variation is intentional.
