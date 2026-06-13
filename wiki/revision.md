# revision

[Back to index](index.md)

The belief revision system in ftl-reasons handles all forms of belief change through two complementary minimal mechanisms — outlist-based defeat for proactive retraction and dependency-directed backtracking for reactive contradiction resolution — together covering the full lifecycle of belief management (`belief-revision-is-comprehensive-and-minimal`). These mechanisms share foundations with the truth maintenance semantics themselves: the same outlist primitive that enables emergent truth evaluation also powers all non-monotonic revision, making minimality a cross-cutting architectural principle rather than a property of any single subsystem (`semantics-and-revision-share-minimal-foundations`).

## Core Mechanisms

The revision system pairs a deterministic reversible reasoning engine with a comprehensive revision layer. The forward direction reliably computes truth states, while the corrective direction handles outlist defeat, contradiction resolution, and dialectical challenge — together forming a complete reasoning-and-revision architecture (`reasoning-and-revision-form-complete-architecture`).

The **outlist primitive** provides a single reversible defeat mechanism that serves multiple roles: challenges, kill-switches, and supersession all operate through the same mechanism. **Dependency-directed backtracking** complements this by resolving detected contradictions, retracting the least-entrenched premise to achieve minimal disruption (`belief-revision-is-comprehensive-and-minimal`). The complete pipeline — outlist defeat for proactive retraction plus backtracking for reactive resolution — produces correct, consistent, auditable results with deterministic propagation that settles all consequences (`belief-revision-is-fully-reliable`).

## Uniform Edge-Case Handling

A notable property of the revision system is that it handles normal beliefs and all edge cases through the same minimal mechanisms. Premises arising from absent justifications, asymmetric missing-node semantics, and vacuously valid empty antecedents all undergo revision through outlist defeat and dependency-directed backtracking — no edge case requires special-case logic (`belief-revision-covers-all-cases-uniformly`).

## Complete Semantics with Controlled Irreversibility

The revision system exhibits complete negative semantics with a controlled asymmetry: all defeat mechanisms — challenge, kill-switch, and supersession — are truth-value reversible, meaning the system can undo the effects of any defeat. However, the identity transformation during challenge, which converts a premise into a justified node, is permanent. The system can restore a defeated node's truth value but cannot restore its original unjustified status (`revision-has-complete-semantics-with-controlled-irreversibility`).

This asymmetry is deterministic and traceable. Every revision operation's outcome is predictable, every effect is auditable, and the distinction between reversible defeat and permanent identity change follows a traceable deterministic path (`revision-semantics-are-deterministic-and-traceable`).

## Rich State Beyond Binary Truth

The revision system governs state far richer than binary IN/OUT truth values. Through metadata-enabled lifecycle governance, revisions track, preserve, and act on retraction reasons, staleness markers, access tags, challenges, and supersession records — state that the binary truth model alone cannot express (`revision-governs-richer-state-than-truth-values`). This richer governance layer operates while remaining exception-safe and recoverable under all failure conditions, ensuring that metadata-carried lifecycle state is never corrupted by exceptions (`revision-is-richly-governed-and-exception-safe`).

## Dialectical Revision

The dialectical revision subsystem achieves three independent trustworthiness properties simultaneously: determinism through semantic transparency inheriting uniform evaluation rules, reliability through safe crash-free premise-to-justified transformation, and semantic completeness with controlled irreversibility (`dialectical-revision-is-deterministic-reliable-and-complete`).

Dialectical revision governs the full metadata-enriched state, producing traceable deterministic changes to retraction flags, stale reasons, and access tags through every challenge/defend operation — not just binary truth transitions (`dialectical-revision-governs-rich-traceable-state`). It operates within an exception-safe framework, ensuring that all dialectical operations produce rich auditable state transitions with safe failure recovery (`dialectical-revision-is-exception-safe-with-rich-traceable-state`).

## Traceability and Recovery

Every revision mechanism — contradiction resolution through dependency-directed backtracking with consistent nogood IDs, and defeat reversal through automatic outlist-driven recovery with surgical restoration hints — provides complete traceability and guided recovery across all revision paths (`all-revision-mechanisms-are-traceable-and-recoverable`).

This extends to exceptional conditions. Whether an operation is normal (outlist defeat, dialectical challenge/defend) or exceptional (contradiction-triggered backtracking, graph inconsistency), the system is simultaneously safe (no crashes or corruption), traceable (deterministic artifact trails), and recoverable (guided restoration hints for cascade victims). The system never enters an unobservable or unrecoverable state regardless of failure mode (`revision-is-exception-safe-and-recoverable`).

## Retracted Claims

Several broader claims about the revision system have been retracted. The claim that revision achieves complete trustworthiness through verifiable soundness, end-to-end reliability, and complete auditability (`revision-achieves-complete-trustworthiness`, OUT) was qualified because revision coverage depends on propagation correctly discovering all dependent nodes — and the dependents index does not track outlist nodes (`revision-coverage-requires-sound-propagation`, OUT). Similarly, claims about universal safety with no blind spots (`revision-is-universally-safe`, OUT) and end-to-end reliability across logical and infrastructure layers (`revision-is-end-to-end-reliable`, OUT) were retracted, as were downstream claims about minimality generating universal revision safety (`minimality-generates-universal-revision-safety`, OUT) and revision completeness following from minimality alone (`revision-completeness-follows-from-minimality`, OUT).

The claim that both revision paths preserve system invariants through shared foundations (`revision-invariants-follow-from-shared-foundations`, OUT) was also retracted, along with its extension to all belief origins (`revision-invariants-span-all-origins`, OUT). Claims about evaluation invariance and auditability across origins (`revision-is-evaluation-invariant-and-auditable-across-origins`, OUT) and the convergence of knowledge revision to self-sustaining equilibria (`knowledge-revision-converges-to-self-sustaining-equilibria`, OUT) are similarly no longer held.

These retractions reflect a pattern: while the core revision mechanisms are well-established and reliable, broader architectural claims about completeness and universality were qualified by the discovery that outlist nodes are not tracked in the dependents index, limiting automatic re-evaluation of certain beliefs when outlist conditions change.
