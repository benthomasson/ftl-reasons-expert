# revision

[Back to index](index.md)

Belief revision in ftl-reasons encompasses all mechanisms by which the system changes its mind — retracting beliefs that become untenable, restoring beliefs when defeats are reversed, and resolving contradictions with minimal disruption. The revision system is built on the same minimal primitives that power truth evaluation itself, achieving comprehensive coverage without special-case logic.

## Core Mechanisms

The system handles all forms of belief revision through two complementary minimal mechanisms (belief-revision-is-comprehensive-and-minimal). The **outlist primitive** provides a single reversible defeat mechanism that serves multiple roles: challenges, kill-switches, and supersession all operate through the same outlist-based retraction. **Dependency-directed backtracking** handles detected contradictions by retracting the least-entrenched premise, ensuring minimal disruption to the rest of the belief network ([contradiction-resolution-is-minimal-disruption](other.md#contradiction-resolution-is-minimal-disruption)).

These two mechanisms — proactive outlist defeat and reactive contradiction resolution — together form a complete revision pipeline that produces correct, consistent, auditable results with deterministic propagation that settles all consequences (belief-revision-is-fully-reliable).

Together with the forward-direction reasoning engine, this creates a complete architecture for belief management: deterministic reversible computation handles truth propagation, while the revision system handles all corrective operations including outlist defeat, contradiction resolution, and dialectical challenge (reasoning-and-revision-form-complete-architecture).

## Uniformity and Minimality

A notable property of the revision system is that it handles normal beliefs and all edge cases — premises arising from absent justifications, asymmetric missing-node semantics, vacuously valid empty antecedents — through the same minimal mechanisms (belief-revision-covers-all-cases-uniformly). No edge case requires special-case logic; outlist defeat and dependency-directed backtracking are sufficient for every scenario.

This minimality is not accidental but reflects a cross-cutting architectural principle. Both truth maintenance semantics and belief revision achieve comprehensive coverage through the same outlist primitive — it simultaneously enables emergent truth evaluation (disjunction over conjunction with absence semantics) and all non-monotonic revision mechanisms (semantics-and-revision-share-minimal-foundations).

## Complete Semantics with Controlled Irreversibility

The revision system exhibits complete negative semantics with a controlled asymmetry: all defeat mechanisms — challenge, kill-switch, supersession — are truth-value reversible, meaning their effects on IN/OUT status can be undone. However, the identity transformation that occurs during challenge (converting a premise to a justified node) is permanent (revision-has-complete-semantics-with-controlled-irreversibility). The system can undo the *effects* of any defeat but cannot restore a node's original unjustified status. This asymmetry is itself deterministic and traceable — every revision operation's outcome is predictable, and the distinction between reversible defeat and permanent identity change follows an auditable path (revision-semantics-are-deterministic-and-traceable).

## Rich State Beyond Binary Truth

Revision operates on more than binary IN/OUT truth values. Through metadata-enabled lifecycle governance, the system tracks, preserves, and acts on richer state including retraction reasons, staleness markers, access tags, challenges, and supersession records (revision-governs-richer-state-than-truth-values). This metadata layer means that a retraction carries not just a truth-value change but an explanation of *why* the belief was retracted and *what* might restore it.

## Dialectical Revision

Dialectical operations — challenge and defend — layer atop the base revision mechanisms and achieve three independent trustworthiness properties simultaneously (dialectical-revision-is-deterministic-reliable-and-complete):

- **Determinism**, inherited from uniform evaluation rules via semantic transparency
- **Reliability**, through safe crash-free transformation of premises to justified nodes
- **Semantic completeness with controlled irreversibility**, where all defeats reverse but identity transformation is permanent

Dialectical revision governs the full metadata-enriched state, producing traceable deterministic changes to retraction flags, stale reasons, and access tags through every challenge/defend operation — not just binary truth transitions (dialectical-revision-governs-rich-traceable-state).

## Exception Safety and Recovery

Every revision mechanism — whether normal (outlist defeat, dialectical challenge/defend) or exceptional (contradiction-triggered backtracking, graph inconsistency) — is simultaneously safe, traceable, and recoverable (revision-is-exception-safe-and-recoverable). The system never enters an unobservable or unrecoverable state regardless of failure mode. Contradiction resolution provides complete traceability through consistent nogood IDs, while defeat reversal offers automatic outlist-driven recovery with surgical restoration hints (all-revision-mechanisms-are-traceable-and-recoverable).

This exception safety composes with rich governance: metadata-carried lifecycle state — retraction reasons, staleness markers, access tags — is never corrupted by exceptions (revision-is-richly-governed-and-exception-safe). Dialectical revision inherits both properties, ensuring that all dialectical operations produce rich auditable state transitions with safe failure recovery across all revision mechanisms (dialectical-revision-is-exception-safe-with-rich-traceable-state).
