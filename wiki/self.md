# self

[Back to index](index.md)

The concept of "self" in ftl-reasons spans several architectural properties — from self-containment (verified and held) to self-correction and self-sustenance (largely aspirational and currently retracted). This is a rich topic with about 70 beliefs. Let me write the page.

## Self-Referential Properties in ftl-reasons

The ftl-reasons codebase exhibits several "self" properties — architectural qualities where the system acts upon or sustains itself without external intervention. These range from verified structural properties like self-containment to aspirational behavioral properties like self-correction and self-sustenance. The distinction matters: the structural claims are well-supported and currently held, while the behavioral claims have been systematically retracted during belief review.

## Self-Containment

The strongest self-referential property is architectural self-containment. The project enforces zero external coupling at both the packaging level (empty `dependencies` in pyproject.toml) and the implementation level (the core Network class uses only stdlib imports), eliminating supply-chain risk entirely (`[project-has-zero-external-coupling](external.md#project-has-zero-external-coupling)`). Internally, despite `network.py` being imported by virtually every module, the three-layer architecture with clean boundaries ensures this central coupling does not create cross-cutting mutation paths (`[central-dependency-is-safely-contained](other.md#central-dependency-is-safely-contained)`). Together, these support the conclusion that the architecture neither imports external risk nor allows internal complexity to leak across layers (`architecture-is-self-contained-and-safely-layered`).

The agent subsystem demonstrates a more localized form of self-containment: it provides complete lifecycle management with isolation through namespace/relay pairs, handles mixed truth states during import, and ensures all defeat operations are reversible for agent reactivation (`[agent-subsystem-is-self-contained](agent.md#agent-subsystem-is-self-contained)`).

## Self-Reinforcing Governance

The governance framework exhibits a recursive quality — governance assurance is itself governed. Hierarchical governance patterns ensure that the mechanisms verifying belief quality are themselves subject to verification, creating a self-reinforcing loop rather than an ungrounded trust assumption (`[governance-assurance-is-universal-and-self-reinforcing](governance.md#governance-assurance-is-universal-and-self-reinforcing)`).

A concrete instance of self-reporting appears in the compaction subsystem: when given a budget, the `compact` output includes a `Token count: N / B budget` line, making resource consumption auditable from within the system itself (`[compact-self-reports-tokens](compact.md#compact-self-reports-tokens)`).

## Self-Correction (Retracted)

A large family of beliefs — over twenty — explored whether ftl-reasons constitutes a self-correcting system. These claims proposed that contradiction resolution through dependency-directed backtracking and staleness detection through source hash comparison would amount to exhaustive, sustainable self-correction operating across the full belief lifecycle.

The most ambitious of these claims asserted that self-correction is exhaustive and self-contained (`self-correction-is-exhaustive-and-self-contained`), requires no external dependencies (`self-correction-requires-no-external-dependencies`), spans both creation and maintenance phases (`self-correction-spans-creation-and-maintenance`), and could sustain the lifecycle indefinitely (`self-correction-sustains-lifecycle-indefinitely`). Related beliefs claimed the self-correction process is fully self-documenting (`self-correction-is-fully-self-documenting`), produces referenceable artifacts (`self-correction-produces-referenceable-artifacts`), and operates within an efficient pipeline (`self-correction-completeness-has-efficient-pipeline`).

All of these beliefs are currently OUT. The root cause traces to `complete-system-is-self-correcting` being marked stale due to invalid scope — the claim overgeneralized from specific mechanisms (backtracking, hash comparison) to a system-wide property that the evidence did not support. When this foundation was retracted, the entire self-correction belief family cascaded to OUT status, demonstrating the TMS's own retraction propagation working as designed.

## Self-Sustaining Systems (Retracted)

A parallel family of beliefs explored self-sustenance — the idea that the system's invariants, once established, maintain themselves as a fixed point. These included claims that minimality is self-sustaining (`minimality-is-self-sustaining`), that invariant preservation is total and self-sustaining (`invariant-preservation-is-total-and-self-sustaining`), and that knowledge revision converges to self-sustaining equilibria (`knowledge-revision-converges-to-self-sustaining-equilibria`).

Higher-order compositions built on these foundations: that the system is a closed self-maintaining architecture (`unified-system-is-a-closed-self-maintaining-architecture`), that self-maintenance is fully auditable (`self-maintenance-is-fully-auditable`), and that self-sustainability is reinforced by resource efficiency (`self-sustainability-is-reinforced-by-resource-efficiency`).

Like the self-correction beliefs, these are all OUT. They represent an aspirational view of the system as a perpetual-motion-like knowledge machine — a view that was not grounded in sufficient evidence about convergence guarantees and resource bounds.

## Trust Boundaries and Self-Maintenance (Retracted)

Several beliefs attempted to bridge self-containment (which is well-supported) with self-maintenance (which is not). The claim that trust boundaries are architecturally enforced (`trust-boundary-is-architecturally-enforced`) combined internal self-containment with external defensive mechanisms (validation pipelines, namespace isolation, agent kill-switches) to argue the system resists corruption from untrusted input. Despite depending on the well-supported self-containment belief, this derived claim is OUT because its other dependency — defensive external containment — was not sufficiently established.

Similarly, `trust-boundaries-are-self-maintaining` and `quality-lifecycle-operates-within-self-maintaining-trust` proposed that trust enforcement is not merely present but self-perpetuating. These remain OUT.

## Pattern: Structural vs. Behavioral Self-Properties

The belief network reveals a clear pattern: **structural** self-properties (self-containment, self-reinforcing governance) survive review, while **behavioral** self-properties (self-correction, self-sustenance, self-maintenance) do not. This distinction likely reflects a difference in verifiability — one can inspect architecture and confirm zero dependencies, but claiming a system corrects itself exhaustively and indefinitely requires proving properties about all possible future states, a much harder standard to meet.

The four IN beliefs about self-properties are grounded in observable code structure. The sixty-plus OUT beliefs extrapolated from those observations toward unbounded behavioral guarantees that the codebase does not yet substantiate.
