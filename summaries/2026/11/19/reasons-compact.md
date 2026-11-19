---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/compact.py

# `reasons/compact.py` — Token-Budgeted Belief State Summary

## Purpose

This module renders a compact, human/LLM-readable snapshot of the entire belief network (RMS = "Reasoning/Reputation Management System" by the surrounding names) that fits inside a token budget. Its reason for existing is **context injection**: the output is designed to be dropped into a `CLAUDE.md` or any LLM context window so an agent can see the current belief state without loading every node.

It owns the presentation strategy only — it does not mutate the network, persist anything, or perform reasoning. It is a pure read/format function over a `Network`.

The output is a markdown-ish list with a fixed section ordering chosen to maximize signal under a tight budget:

1. **Nogoods** — contradiction records (highest signal, first to consume budget)
2. **OUT (retracted)** — retracted/stale/superseded beliefs that need review
3. **IN (active)** — live beliefs, ordered by how many things depend on them

## Key Components

### `estimate_tokens(text: str) -> int`
```python
return max(1, len(text) // 4)
```
The standard "4 chars ≈ 1 BPE token" heuristic. The `max(1, ...)` guarantees a non-zero estimate even for empty strings, avoiding a `_current_tokens()` of 0 that would break the `> budget` comparisons for degenerate inputs.

### `compact(network, budget=500, truncate=True) -> str`
The single public entry point.

| Parameter | Meaning |
|---|---|
| `network` | A `Network` instance; used duck-typed (only `.nodes`, `.nogoods` are touched) |
| `budget` | Estimated token ceiling for the whole rendered string. Default `500` |
| `truncate` | If `True`, node **text** is clipped to 77 chars + `"..."` (80 total) via `_text`. Does **not** clip nogood IDs, deps lists, or reason strings |

Returns a `"\n"`-joined string, always ending with a `Token count: ~N / budget budget` footer.

### Internal helpers (closures)
- `_text(node)` — clips `node.text` to 80 chars when `truncate` is set.
- `_char_count` — nonlocal running character count of everything appended so far, so token totals are `O(1)` per line instead of re-joining `lines` repeatedly.
- `_add_line(line)` — appends a line and bumps `_char_count` by `len(line) + 1` for the newline separator.
- `_current_tokens()` — `max(1, _char_count // 4)`.
- `_over_budget(line)` — `_current_tokens() + estimate_tokens(line) + footer_tokens > budget`. Note this is a **look-ahead** test: it asks "would adding this line (plus reserving the footer) blow the budget?"

## Patterns

- **Priority-ordered, budget-shared sections.** Every section competes for the same `budget`; there is no per-section quota. If nogoods are numerous/long, OUT and IN get nothing. This is the central design tradeoff.
- **Look-ahead budget reservation.** `_over_budget` reserves `footer_tokens` on every check so the closing token-count line is guaranteed to fit. The footer's own length is estimated from a max-width `budget` placeholder (`f"Token count: ~{budget} / {budget} budget"`), which is a conservative upper bound.
- **Best-effort truncation.** Rather than raising, each section emits a `... (N more … omitted)` line on overflow and `break`s. Truncation is silent and structural, not an error.
- **Closure-based accumulator.** `_char_count` / `_add_line` avoid repeatedly re-joining the whole string.
- **Summary deduplication.** Nodes carrying `metadata["summarizes"]` act as rollups: any IN node whose id appears in another node's `summarizes` list is suppressed from the IN listing and counted in `hidden_count`. This lets the caller maintain summary nodes to keep the scene compact.
- **Provenance reason surfacing.** OUT nodes get a suffix derived from `retract_reason` → `stale_reason` → `superseded_by` (in priority order), so the reader sees *why* a belief was retracted.

## Dependencies

**Imports:**
- `datetime.date` — used once for the header date stamp (`date.today().isoformat()`).
- `Network` from `.network` — only a type hint; runtime behavior is duck-typed against `network.nodes` (dict of id → node) and `network.nogoods` (iterable of nogood objects).

**Imported by:**
- `reasons/api.py` — re-exported as part of the public API surface.
- `reasons_lib/compact.py` — the parallel `reasons_lib/` mirror package (the repo has both; likely one is a shim/re-export over the other).
- `tests/test_compact.py`, `tests/test_summarize.py` — behavior contracts.

## Flow

1. **Partition nodes.** Iterate `network.nodes.values()`; `truth_value == "IN"` goes to `in_nodes`, everything else to `out_nodes`. (Any truth value other than `"IN"` is treated as OUT.)
2. **Sort IN nodes** by `len(node.dependents)` descending — the initial sort is later effectively re-done when `visible_nodes` is sorted; the earlier sort's practical effect is minimal.
3. **Header block:** date, `total | nogoods | IN | OUT` counts, blank line.
4. **Initialize budget tracker.** `footer_tokens` is computed from a placeholder footer string, and `_char_count` is seeded from the header lines `+ len(lines) - 1` for the `\n` separators.
5. **Section 1 — Nogoods.** Only enters if `_over_budget("## Nogoods")` is false. Each nogood renders as `- {id}: {comma-joined node ids} — {resolution?}`. On overflow, emits `... (N more nogoods omitted)` and breaks.
6. **Section 2 — OUT.** Same structure. Each line is `- {id}: {text} [(stale: …)/ (superseded by: …)]`. `retract_reason`/`stale_reason` is preferred over `superseded_by`. Reason strings are truncated to 60 chars inline.
7. **Section 3 — IN.** Splits `in_nodes` into `summary_nodes` (those with `metadata["summarizes"]`) and `regular_nodes`. Builds `covered_by_summary` from every summary's covered IDs. `visible_nodes = summary_nodes + [regular not covered]`, then re-sorted by dependent count. Each line includes:
   - `[summary]` prefix when applicable
   - first non-`summarizes` justification's antecedents as `<- a, b, c`
   - `(N dependents)` when > 0
   - `(covers N nodes)` when it's a summary
8. **Hidden count line** — if summaries hid any nodes, appends `  (N nodes hidden by summaries)`. This is emitted **even if the section already overflowed and broke**, so both the omission line and the hidden line can appear.
9. **Footer** — `Token count: ~{_current_tokens()} / {budget} budget`. Because the f-string evaluates `_current_tokens()` *before* the line is appended and doesn't call `_add_line`, the reported number excludes the footer itself (correctly representing "used so far").

## Invariants

- `estimate_tokens` is always ≥ 1.
- When `truncate=True`, any node's rendered text is at most **77 + 3 = 80** chars.
- The reported `Token count` reflects `_char_count // 4` at the moment just before the footer is appended, so it does **not** include the footer's own characters.
- The header block and the footer are **always emitted**, regardless of budget — a very small `budget` cannot suppress them; only the three content sections are gated by `_over_budget`.
- Sections appear in strict priority order: nogoods → OUT → IN. There is no interleaving.
- `hide` semantics: an IN node is hidden **iff** its id appears in the `summarizes` list of some *other* IN node. Summary nodes themselves are never hidden.
- The `_over_budget` guard treats the reserved footer as always-present, so the actual overrun (if any) is only ever caused by the unguarded omission/hidden lines.
- The module never mutates `network`, `node.metadata`, or any node attributes.

## Error Handling

There is **no explicit error handling or validation**. The function assumes:
- `network.nodes` is a mapping of id → node
- `network.nogoods` is iterable
- Every node exposes `.truth_value`, `.text`, `.id`, `.metadata` (dict), `.dependents` (with `len`), `.justifications` (each with `.antecedents` and `.label`)
- Every nogood exposes `.id`, `.nodes`, `.resolution`

A missing attribute raises `AttributeError` and propagates to the caller; a malformed `truth_value` is silently classified as OUT. There are no `try/except` blocks, no logging, and no warnings for budget overrun — the only signal is the footer token line and the `... (N more … omitted)` markers.

The best-effort overflow path is intentional: it prefers a *slightly* over-budget string with an omission marker over raising or truncating mid-line.

## Topics to Explore

- [file] `reasons/network.py` — Defines the `Network` and node types this function reads (`truth_value`, `dependents`, `justifications`, `metadata`, `nogoods`); the real contracts live here, not in `compact.py`.
- [file] `reasons/api.py` — Shows how `compact` is surfaced in the public Python API and whether it is wrapped/re-exported.
- [general] `reasons-lib-mirror` — The repo contains both `reasons/` and `reasons_lib/`; understand which is authoritative and whether `reasons_lib/compact.py` is a copy, a shim, or a build artifact, since both are imported by tests.
- [file] `tests/test_summarize.py` — Exercises the `metadata["summarizes"]` rollup path — the most subtle piece of logic in this file (dedup + `hidden_count`).
- [file] `tests/test_compact.py` — Pins the exact budget/truncation/footer behavior; useful for sanity-checking any change to the `_over_budget` arithmetic.

## Beliefs

- `token-estimate-uses-chars-div-4` — `estimate_tokens` returns `max(1, len(text) // 4)`, the standard BPE approximation.
- `compact-prioritizes-nogoods-then-out-then-in` — Sections are emitted in that order and share a single `budget`, so nogoods can exhaust the budget before OUT/IN appear.
- `budget-check-reserves-footer-space` — `_over_budget` adds `footer_tokens` to every line check so the closing token-count line always has reserved room.
- `summaries-hide-covering-in-nodes` — An IN node whose id appears in another IN node's `metadata["summarizes"]` is omitted from the IN listing and reported via `(N nodes hidden by summaries)`.
- `truncate-defaults-to-80-char-node-text` — With `truncate=True` (default), `_text` clips node text to 77 chars plus `"..."`, i.e. 80 total.
- `omission-and-hidden-lines-bypass-budget-guard` — The `... (N more … omitted)` and `(N nodes hidden by summaries)` lines are appended without `_over_budget` checks, so a run can exceed `budget` slightly.
