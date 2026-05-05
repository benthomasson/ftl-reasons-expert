# Epistemology of Derived Beliefs

**Date:** 2026-05-05

## The Question

How do we know that beliefs and derived conclusions in the network are true?

## Current Assurance Layers

The system provides several layers of assurance, but they are uneven in strength:

### Premises (strongest)

Premises are direct observations from source code. They are grounded in what the code actually does, not in reasoning about what it should do. The `check-stale` mechanism can verify premises against the current codebase by re-reading the source file and line range recorded at assertion time. When code changes invalidate a premise, staleness detection flags it for review or retraction.

### Structural Validation (mechanical)

Derived beliefs must pass structural checks before acceptance:

- All referenced antecedent beliefs must exist and be IN
- Cycle and depth guards prevent infinite derivation chains
- Jaccard similarity guards prevent re-derivation of previously retracted conclusions
- Budget limits cap the number of derivations per round
- The derive prompt roundtrips through a parser that enforces format contracts

These checks ensure the justification *structure* is valid but say nothing about whether the *reasoning* is sound.

### Review Gates (human + LLM)

The `code-expert review` step filters proposals before acceptance. In the most recent update, it rejected 50 of 78 proposals (49 duplicates, 1 trivial). This catches redundancy and low-value beliefs but does not deeply verify semantic correctness.

### Multi-Model Code Review (indirect)

Code reviews using multiple models (Claude + Gemini) catch issues in the code that beliefs describe, which can surface incorrect premises. But this is indirect -- it validates the code, not the beliefs about the code.

## The Gap

Derived beliefs are semantically validated only by the LLM that proposed them. The system checks that justification references are valid, but not that the logical step from antecedents to conclusion is sound. A plausible-sounding but incorrect conclusion gets accepted if its structural references check out.

At depth 15 with 437 derived beliefs, reasoning errors can compound: if belief A is subtly wrong and belief B is derived from A, B inherits A's error and may amplify it.

## The Answer

Beliefs are **tentatively true according to the decisions made by the LLM, given the information it had at the time.**

This is not a weakness -- it is the designed operating mode of a Truth Maintenance System. The whole point of Doyle's 1979 TMS architecture is that beliefs are not permanently true. They are held because their justifications currently support them. The system provides:

1. **Traceability** -- every belief records *why* it is held (its justification chain back to premises)
2. **Revisability** -- when support changes, dependent beliefs are automatically retracted via dependency-directed backtracking
3. **Non-monotonicity** -- outlist mechanisms allow beliefs to be held precisely because other beliefs are *not* held, enabling default reasoning that gracefully retracts when exceptions appear

The value proposition is not certainty but **traceable, revisable reasoning**. You can always ask "why is this believed?" (`reasons explain <id>`) and get a complete answer. You can always retract a premise and watch the system automatically propagate the consequences. You can record contradictions as nogoods and the system will retract the least-entrenched contributor.

## Comparison to Alternative Approaches

A system that only stored "verified facts" would be smaller and more confident but also more brittle -- it could not represent tentative conclusions, default assumptions, or reasoning under uncertainty. It would have no mechanism for graceful retraction when new information arrives.

A system with no justification tracking (a flat knowledge base) would have no way to answer "why do you believe this?" and no way to automatically retract dependent conclusions when a foundation belief is invalidated.

The TMS approach occupies a middle ground: beliefs are held with explicit justification, automatically maintained, and always subject to revision. The epistemological status of each belief is transparent and mechanically traceable.
