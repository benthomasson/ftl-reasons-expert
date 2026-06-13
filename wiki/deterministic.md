# deterministic

[Back to index](index.md)

Determinism is a foundational property of the ftl-reasons Truth Maintenance System, ensuring that truth evaluation, state transitions, and output production yield predictable, reproducible results. From the core engine's uniform evaluation rules through dialectical operations, contradiction resolution, and output formatting, the system is designed so that identical inputs always produce identical outcomes — and every path to those outcomes is traceable.

## Core Engine

The TMS engine achieves deterministic, terminating truth maintenance through three reinforcing properties: uniform pure evaluation of justifications, guaranteed convergence of propagation, and conservative asymmetric failure semantics for missing nodes (`tms-core-is-deterministic-and-conservative`). Built on this foundation, the reasoning engine extends determinism to non-monotonic operations — challenge, kill-switch, supersession, and dialectics are all inherently reversible through a single outlist primitive (`reasoning-engine-is-deterministic-and-reversible`). This combination means that any given set of premises produces exactly one truth-value assignment.

When paired with gapless lifecycle management, the result is a system whose belief-state trajectory is both fully determined and fully monitored: no belief escapes tracking across any lifecycle phase, and the state at any point is predictable from its inputs (`deterministic-reasoning-with-gapless-lifecycle`). Bidirectional modification — contradiction resolution through backtracking and defeat reversal with guided recovery — operates within this deterministic lifecycle framework (`bidirectional-modification-within-deterministic-lifecycle`).

## Dialectical Operations

Dialectical challenge and defend operations inherit their determinism rather than implementing it independently. Because dialectical structures are semantically transparent — the engine treats dialectical nodes identically to ordinary beliefs — they receive deterministic reversible evaluation without special-casing (`dialectics-are-deterministic-by-transparency`). This transparency makes dialectics simultaneously deterministic and fully reliable, with semantics-preserving crash safety through terminating propagation (`dialectics-are-deterministic-and-reliable`).

The grounding runs deeper than transparency alone. Complete reversible negative semantics — structural absence producing emergent premise behavior, plus explicit outlist defeat with automatic reversal — are the foundation that enables deterministic dialectics. Challenge and defend operations inherit determinism from evaluation purity applied to outlist primitives, and reliability from the inherent reversibility of outlist-based defeat (`negative-semantics-ground-deterministic-dialectics`). This grounding is further reinforced transitively: uniform edge-case handling ensures that all semantic edge cases (vacuous premises, asymmetric absence, empty antecedents) follow the same rules that produce outlist defeat, which in turn strengthens the negative semantics that ground dialectics (`uniform-semantics-transitively-ground-deterministic-dialectics`).

## Contradiction Resolution

When contradictions are detected, resolution follows a deterministic pipeline. Backtracking identifies the least-entrenched culprit premise, retraction triggers BFS propagation that terminates via stop-on-unchanged, and the system converges to a new consistent state with minimal network disruption (`contradiction-triggers-deterministic-resolution`). Propagation termination is guaranteed by the core engine's convergence properties, which this pipeline inherits.

Interestingly, the contradiction *detection* phase deliberately introduces controlled non-determinism: belief IDs are randomly shuffled before batching so that repeated runs cover different pairwise combinations across batch boundaries, increasing cross-batch contradiction coverage (`contradictions-shuffle-prevents-deterministic-batching`). This is a pragmatic design choice — detection sampling varies, but once a contradiction is found, resolution is fully deterministic.

## State Transitions and Traceability

Every belief state change — whether initiated by intentional dialectical challenge/defend or by automated contradiction resolution — follows a deterministic evaluation path and produces a complete traceable history (`all-state-transitions-are-deterministic-and-traceable`). No state transition is opaque or unpredictable. This belief underpins several broader claims about system-wide traceability and revision semantics.

A more ambitious claim that all belief transformations — including identity transformations like irreversible premise-to-justified conversion — are simultaneously deterministic, traceable, and boundary-safe has been retracted (`all-transformations-are-deterministic-traceable-and-boundary-safe`), as have related beliefs about deterministic reasoning operating within evolution-tolerant boundaries (`deterministic-reasoning-within-evolution-tolerant-boundaries`) and achieving boundary-safe reproducibility (`deterministic-reasoning-is-boundary-safe-and-reproducible`). The core state-transition determinism holds, but extending it to boundary safety and identity transformation guarantees is no longer maintained.

## Derivation Reproducibility

The derive pipeline ensures reproducible LLM-driven derivation through deterministic prompt construction. Beliefs are sorted by ID before embedding, making cluster assignments reproducible given the same random seed (`cluster-embed-order-is-deterministic`). Cluster-based belief selection produces identical results for a given seed and returns exactly the requested budget count (`cluster-selection-is-deterministic-and-budget-exact`). In sample mode, `_build_beliefs_section` with a fixed seed produces identical output across calls (`sample-mode-is-deterministic`). Together, these ensure that derive prompts are fully reproducible — although LLM output itself remains stochastic, the inputs to derivation are controlled.

## CLI and Output Determinism

The CLI achieves full scriptability through three deterministic properties: flat dict dispatch with no dynamic plugin resolution, binary exit codes (0 success, 1 error) with no ambiguous intermediate codes, and clean stream separation with diagnostics to stderr and results to stdout (`cli-is-deterministic-and-stream-correct`).

Staleness checking produces deterministic output sorted by node ID, with a uniform 6-key schema across all result types and exception-free handling of missing files (`check-stale-output-is-deterministic-and-structured`). This makes staleness results suitable for programmatic consumption and diffing.

Several broader output-determinism beliefs have been retracted, including claims that all read paths are deterministic and resilient (`all-read-paths-are-deterministic-and-resilient`), that output is simultaneously deterministic, authorized, and resilient (`all-output-is-deterministic-authorized-and-resilient`), and that query degradation maintains determinism across all access paths (`query-degradation-is-deterministic-across-all-access-paths`). The determinism of individual output paths like staleness checking and the CLI remains well-established, but system-wide output determinism guarantees are no longer held.

## Cross-Origin Determinism

Claims that deterministic revision applies uniformly across all belief origins — human-initiated, LLM-derived, and multi-agent imported — have been retracted (`all-belief-origins-share-deterministic-revision`), as has the stronger claim that deterministic traceable history extends to externally-originated beliefs at full integration parity (`deterministic-history-extends-to-all-origins`). While the core engine evaluates all beliefs through the same deterministic mechanism regardless of origin, the broader guarantees about uniform treatment across all origin types and complete cross-origin traceability are no longer maintained as active beliefs.
