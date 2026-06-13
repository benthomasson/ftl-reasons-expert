# compact

[Back to index](index.md)

### compact-budget-controls-output-size
**Status:** OUT

The compact module's token budget reliably constrains total output size

**Depends on:** [compact-in-nodes-ordered-by-dependents](compact.md#compact-in-nodes-ordered-by-dependents), [compact-summary-hiding-requires-in](compact.md#compact-summary-hiding-requires-in)
**Supports:** [compact-is-predictable-bounded-distillation](compact.md#compact-is-predictable-bounded-distillation), [external-interface-is-bidirectionally-bounded](external.md#external-interface-is-bidirectionally-bounded), [information-flow-is-authorization-and-budget-controlled](other.md#information-flow-is-authorization-and-budget-controlled), [token-budgets-are-accurate-bidirectionally](other.md#token-budgets-are-accurate-bidirectionally)

### compact-budget-guarantee
**Status:** OUT

`compact()` output estimated token count never exceeds the `budget` parameter; every line is gated by `_over_budget` before appending.


### compact-budget-is-soft-cap
**Status:** IN

The compact token budget is approximate; structural overhead (section headers, truncation messages) can cause up to ~25% overshoot beyond the specified budget.


### compact-budget-only-limits-in-nodes
**Status:** OUT

The token budget only constrains the IN nodes section; nogoods and OUT nodes are always emitted regardless of budget, so compact output can exceed the specified budget value.

**Supports:** [compact-budget-controls-output-size](compact.md#compact-budget-controls-output-size)

### compact-budget-soft-ceiling
**Status:** IN

The `compact` function treats the token budget as approximate; structural overhead (headers, truncation messages) can cause output to exceed the budget by up to ~25%.


### compact-budget-tracking-is-efficient-and-approximate
**Status:** IN

The compact module tracks token budgets efficiently through an approximate but computationally fast strategy: O(1) per-line budget checks via a running character count, with token estimation based on chars/4 — a lightweight approximation avoiding external tokenizer dependencies while maintaining accuracy sufficient for budget enforcement.

**Depends on:** [compact-char-tracking-o1](compact.md#compact-char-tracking-o1), [compact-estimate-tokens-chars-div-4](compact.md#compact-estimate-tokens-chars-div-4)
**Supports:** [budget-enforcement-is-efficient-across-pipeline](other.md#budget-enforcement-is-efficient-across-pipeline), [compact-is-efficient-deterministic-and-bounded](compact.md#compact-is-efficient-deterministic-and-bounded)

### compact-char-tracking-o1
**Status:** IN

Budget tracking uses a running `_char_count` integer, making per-line budget checks O(1) instead of O(n).

**Supports:** [compact-budget-tracking-is-efficient-and-approximate](compact.md#compact-budget-tracking-is-efficient-and-approximate)

### compact-estimate-tokens-chars-div-4
**Status:** IN

`estimate_tokens` uses `len(text) // 4` with a floor of 1, not word count or any external tokenizer.

**Supports:** [compact-budget-tracking-is-efficient-and-approximate](compact.md#compact-budget-tracking-is-efficient-and-approximate)

### compact-footer-pre-reserved
**Status:** IN

The footer line's token cost is pre-computed and reserved before any section starts filling, guaranteeing it always fits within the budget.

**Supports:** [compact-structure-is-priority-ordered-and-infallible](compact.md#compact-structure-is-priority-ordered-and-infallible)

### compact-in-nodes-ordered-by-dependents
**Status:** IN

IN nodes are sorted by descending dependent count so structurally important nodes (those depended on by many others) are emitted first and survive budget truncation.

**Supports:** [compact-budget-controls-output-size](compact.md#compact-budget-controls-output-size)

### compact-is-deterministic-pure-and-bounded
**Status:** OUT

The compact module produces output that is simultaneously deterministic (pure function with fixed priority ordering), bounded (guaranteed to never exceed the token budget), and self-describing (includes its own token count for auditability).

**Depends on:** [compact-is-pure-function](compact.md#compact-is-pure-function), [compact-never-exceeds-budget](compact.md#compact-never-exceeds-budget), [compact-self-reports-tokens](compact.md#compact-self-reports-tokens)
**Supports:** [compact-is-efficient-deterministic-and-bounded](compact.md#compact-is-efficient-deterministic-and-bounded), [compact-is-predictable-bounded-distillation](compact.md#compact-is-predictable-bounded-distillation)

### compact-is-efficient-deterministic-and-bounded
**Status:** OUT

The compact module simultaneously achieves computational efficiency (O(1) per-line budget tracking via running character count with chars/4 token estimation), mathematical determinism (pure function with no side effects), and guaranteed output bounds (never exceeds the budget parameter) — all three desirable output properties without trade-offs.

**Depends on:** [compact-budget-tracking-is-efficient-and-approximate](compact.md#compact-budget-tracking-is-efficient-and-approximate), [compact-is-deterministic-pure-and-bounded](compact.md#compact-is-deterministic-pure-and-bounded)
**Supports:** [system-efficiency-spans-packaging-and-runtime](system.md#system-efficiency-spans-packaging-and-runtime)

### compact-is-infallible
**Status:** IN

`compact()` handles empty networks, zero-budget, and missing metadata without raising exceptions — designed to always produce a valid string.

**Supports:** [compact-structure-is-priority-ordered-and-infallible](compact.md#compact-structure-is-priority-ordered-and-infallible)

### compact-is-predictable-bounded-distillation
**Status:** OUT

The compact module is a fully predictable information distillation: a pure function with deterministic priority ordering that reliably constrains output within token budgets, self-reports resource usage, and structurally important nodes are prioritized through dependent-count sorting

**Depends on:** [compact-budget-controls-output-size](compact.md#compact-budget-controls-output-size), [compact-is-deterministic-pure-and-bounded](compact.md#compact-is-deterministic-pure-and-bounded)
**Supports:** [all-read-paths-are-deterministic-and-resilient](deterministic.md#all-read-paths-are-deterministic-and-resilient), [compact-output-is-structurally-complete-and-predictably-bounded](compact.md#compact-output-is-structurally-complete-and-predictably-bounded), [information-output-is-authorized-budgeted-and-deterministic](deterministic.md#information-output-is-authorized-budgeted-and-deterministic)

### compact-is-pure-function
**Status:** IN

`compact()` performs no I/O, no database access, no mutations, and no side effects; it is a pure transformation from `Network` to `str`.

**Supports:** [compact-is-deterministic-pure-and-bounded](compact.md#compact-is-deterministic-pure-and-bounded)

### compact-never-exceeds-budget
**Status:** IN

`compact()` guarantees the returned string's estimated token count (chars/4) does not exceed the `budget` parameter, enforced by pre-checking every line addition against remaining space.

**Supports:** [compact-is-deterministic-pure-and-bounded](compact.md#compact-is-deterministic-pure-and-bounded)

### compact-output-is-structurally-complete-and-predictably-bounded
**Status:** OUT

The compact module simultaneously achieves structural completeness — priority-ordered sections where later sections never displace earlier ones, infallible handling of all edge cases including empty networks and zero budgets, and pre-reserved footer guaranteeing auditability metadata — and predictable resource bounding through a pure deterministic function with guaranteed budget enforcement and self-reporting token counts, making compact a reliable building block for automated context-limited pipelines.

**Depends on:** [compact-is-predictable-bounded-distillation](compact.md#compact-is-predictable-bounded-distillation), [compact-structure-is-priority-ordered-and-infallible](compact.md#compact-structure-is-priority-ordered-and-infallible)
**Supports:** [system-output-is-complete-bounded-and-ci-ready](system.md#system-output-is-complete-bounded-and-ci-ready)

### compact-priority-order
**Status:** IN

Sections are emitted in fixed order: nogoods, OUT nodes, IN nodes; a later section never displaces content from an earlier one.

**Supports:** [compact-structure-is-priority-ordered-and-infallible](compact.md#compact-structure-is-priority-ordered-and-infallible)

### compact-priority-order-is-nogoods-out-in
**Status:** IN

Sections are emitted in strict priority order: nogoods first, then OUT nodes, then IN nodes; if the budget is exhausted early, lower-priority sections are entirely omitted.


### compact-pure-function
**Status:** IN

`compact()` has no side effects, performs no I/O, and does not mutate the `Network` instance.


### compact-self-reports-tokens
**Status:** IN

When given a budget, the `compact` output includes a `Token count: N / B budget` line for auditability.

**Supports:** [compact-is-deterministic-pure-and-bounded](compact.md#compact-is-deterministic-pure-and-bounded)

### compact-sorts-by-dependents
**Status:** IN

Duplicate of existing `compact-in-nodes-ordered-by-dependents`.


### compact-sorts-in-nodes-by-dependents-descending
**Status:** IN

Duplicates existing belief `compact-in-nodes-ordered-by-dependents`.


### compact-structure-is-priority-ordered-and-infallible
**Status:** IN

The compact module produces output with deterministic priority-ordered sections (nogoods, OUT, IN) and handles all edge cases (empty networks, zero budget, missing metadata) without raising exceptions, with footer space pre-reserved before section filling begins.

**Depends on:** [compact-footer-pre-reserved](compact.md#compact-footer-pre-reserved), [compact-is-infallible](compact.md#compact-is-infallible), [compact-priority-order](compact.md#compact-priority-order)
**Supports:** [compact-output-is-structurally-complete-and-predictably-bounded](compact.md#compact-output-is-structurally-complete-and-predictably-bounded)

### compact-summary-hiding-requires-in
**Status:** IN

A summary node only hides its covered nodes when the summary itself is IN; if the summary goes OUT, covered nodes reappear in the compact output.

**Supports:** [compact-budget-controls-output-size](compact.md#compact-budget-controls-output-size)

### compact-summary-nodes-hide-covered
**Status:** IN

IN nodes whose IDs appear in another node's `summarizes` metadata list are excluded from compact output to avoid redundancy; the hidden count is reported.


### compact-surfaces-stale-reason-metadata
**Status:** IN

OUT nodes with `stale_reason` in their metadata include the reason string in the compact output.

**Supports:** [staleness-information-survives-binary-truth-model](other.md#staleness-information-survives-binary-truth-model)

### compact-three-sections
**Status:** IN

The `compact` output is organized into three markdown sections: `## IN (active)`, `## OUT (retracted)`, and `## Nogoods`, each present only when the network contains nodes of that type.


### compact-token-estimate-is-word-count
**Status:** OUT

`estimate_tokens` counts whitespace-separated words, not BPE tokens; the budget parameter throughout the compact module is measured in this unit.

**Supports:** [compact-budget-controls-output-size](compact.md#compact-budget-controls-output-size)

### pg-compact-generates-budget-constrained-markdown
**Status:** IN

`PgApi.compact()` produces a markdown summary of the belief network constrained by a token budget, filtered by `visible_to` access tags, sorted by dependent count, with summary-node elision and nogood inclusion

