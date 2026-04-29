# File: reasons_lib/compact.py

**Date:** 2026-04-29
**Time:** 17:07

## Purpose

`compact.py` owns one responsibility: producing a **token-budgeted plain-text summary** of the belief network's current state. The output is designed for injection into LLM context windows (CLAUDE.md files, system prompts, etc.) where token real estate is limited. It answers "what does the belief network look like right now?" in the fewest tokens possible, prioritizing the most actionable information — contradictions first, then problems, then healthy state.

## Key Components

### `estimate_tokens(text: str) -> int`

A cheap heuristic: `len(text) // 4`, floored at 1. This approximates BPE tokenization without pulling in a real tokenizer. It's used everywhere in the budget accounting — the entire file's budget logic depends on this single approximation.

### `compact(network: Network, budget: int = 500, truncate: bool = True) -> str`

The main entry point. Takes a `Network` (the in-memory belief graph), a token budget (default 500), and a truncation flag. Returns a multi-section markdown string that fits within the budget.

**Contract**: the returned string's estimated token count will not exceed `budget`. The function never raises — it gracefully degrades by omitting nodes when space runs out.

## Patterns

**Priority-ordered greedy packing.** This is essentially a knapsack-style greedy fill: sections are emitted in strict priority order (nogoods → OUT → IN), and within each section, items are added one by one until the budget is exhausted. When an item would blow the budget, a `... (N more omitted)` sentinel is appended and the section ends.

**Running character count instead of re-joining.** Rather than calling `"\n".join(lines)` and re-estimating tokens on every iteration (O(n²)), the code maintains `_char_count` as a running total and derives tokens from it in O(1) via `_current_tokens()`. The closures `_add_line`, `_current_tokens`, and `_over_budget` form a mini state machine around this counter.

**Summary deduplication.** In the IN section, nodes with a `summarizes` metadata key are treated as summaries that cover other nodes. Covered nodes are hidden from the output to avoid redundancy, and the hidden count is reported at the end. Summary nodes sort to the top alongside other high-dependent-count nodes.

## Dependencies

**Imports:**
- `datetime.date` — for the ISO date stamp in the header
- `.network.Network` — the core belief graph data structure (nodes, nogoods, justifications)

**Imported by:**
- `reasons_lib/api.py` — the public API surface exposes `compact` to CLI and external callers
- `tests/test_compact.py` and `tests/test_summarize.py` — unit tests

The module has no I/O, no database access, and no side effects. It's a pure function over the `Network` data structure.

## Flow

1. **Partition** all nodes into `in_nodes` and `out_nodes` by truth value.
2. **Sort** `in_nodes` by dependent count descending (most-depended-on first — these are the "load-bearing" beliefs).
3. **Emit header** with date, total counts, and nogood count.
4. **Reserve footer space** by pre-computing `footer_tokens`.
5. **Fill Section 1 (Nogoods)**: iterate `network.nogoods`, emit `id: node-list [resolution]`. Stop when budget would overflow.
6. **Fill Section 2 (OUT nodes)**: iterate `out_nodes`, annotate with `retract_reason`, `stale_reason`, or `superseded_by` metadata. Stop at budget.
7. **Fill Section 3 (IN nodes)**: first separate summary nodes from regular nodes, build a `covered_by_summary` set, then merge them into `visible_nodes` (summaries + uncovered regulars), re-sort by dependent count. Emit with justification chains (`<- antecedents`), dependent count, and summary coverage info. Stop at budget.
8. **Append footer** with final token count vs. budget.
9. **Join and return.**

## Invariants

- **Budget is never exceeded.** Every line addition is guarded by `_over_budget()`, which checks `current_tokens + line_tokens + footer_tokens <= budget`.
- **Priority ordering is strict.** Nogoods are always emitted before OUT nodes, which are always emitted before IN nodes. If the budget is tiny, lower-priority sections are entirely omitted.
- **The footer is always present.** Footer tokens are reserved upfront, so the final `Token count:` line always fits.
- **`estimate_tokens` returns at least 1.** The `max(1, ...)` prevents division-by-zero or zero-token accounting for empty strings.
- **IN nodes are sorted by influence.** Within the IN section, nodes with more dependents appear first, meaning if the budget cuts off the section, the most structurally important beliefs survive.

## Error Handling

There is effectively none — and intentionally so. The function operates over an already-validated `Network` object. No I/O, no parsing, no user input. If `network.nodes` is empty, it produces a valid summary with just the header and footer. Missing metadata keys are handled with `.get()` defaulting to `None`/falsy, so nodes without `retract_reason` or `summarizes` metadata are handled gracefully.

## Topics to Explore

- [file] `reasons_lib/network.py` — The `Network`, `Node`, and `Nogood` data structures that `compact` reads from — understanding their fields (especially `dependents`, `justifications`, `metadata`) is essential to understanding what compact renders
- [file] `tests/test_compact.py` — Shows the expected output format and edge cases (empty networks, budget overflow, truncation behavior)
- [file] `tests/test_summarize.py` — Tests the summary-node deduplication logic specifically — how `summarizes` metadata interacts with the IN section
- [function] `reasons_lib/api.py:compact` — How the CLI/API layer calls this function and what `budget`/`truncate` defaults are used in practice
- [general] `token-budget-accuracy` — The chars/4 heuristic can drift significantly from real BPE token counts, especially with code-heavy or non-English text; worth validating if budgets are tight

## Beliefs

- `compact-never-exceeds-budget` — `compact()` guarantees the returned string's estimated token count (chars/4) does not exceed the `budget` parameter, enforced by pre-checking every line addition
- `compact-priority-order-is-nogoods-out-in` — Sections are emitted in strict priority order: nogoods first, then OUT nodes, then IN nodes; lower-priority sections are entirely dropped if budget is exhausted
- `compact-is-pure-function` — `compact()` performs no I/O, no mutations, and no side effects; it is a pure transformation from `Network` to `str`
- `compact-summary-nodes-hide-covered` — IN nodes whose IDs appear in another node's `summarizes` metadata list are excluded from the output to avoid redundancy
- `compact-in-nodes-sorted-by-dependent-count` — IN nodes are sorted by `len(node.dependents)` descending, so structurally important beliefs survive budget truncation

