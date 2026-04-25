# File: reasons_lib/compact.py

**Date:** 2026-04-24
**Time:** 17:02

## Purpose

`compact.py` owns **token-budgeted summarization** of the belief network. Its job is to produce a text snapshot of the entire network state that fits within a caller-specified token budget — designed for injection into LLM context windows (e.g., CLAUDE.md files or system prompts). It's the bridge between the full graph in SQLite and the constrained context an LLM actually sees.

The module exports a single public function (`compact`) and one helper (`estimate_tokens`).

## Key Components

### `estimate_tokens(text: str) -> int`
Approximates token count using `len(text) // 4`, the standard BPE heuristic. Returns at least 1 (never zero). This is a coarse estimate — it's intentionally cheap since it's called per-line during budget tracking.

### `compact(network, budget=500, truncate=True) -> str`
The main function. Takes a `Network` instance and returns a markdown-formatted string that fits within `budget` tokens. The `truncate` flag controls whether individual node text is clipped to 80 characters.

**Contract**: The returned string's estimated token count will not exceed `budget`. The function never raises — it gracefully degrades by omitting nodes with "... (N more omitted)" messages.

## Patterns

**Priority-based greedy packing.** The function fills a fixed budget in strict priority order: nogoods first, then OUT nodes, then IN nodes. Each section checks `_over_budget` before adding every line, and emits a truncation marker when the budget is exhausted. This is a greedy bin-packing approach — no backtracking or rebalancing.

**Running character counter.** Instead of rejoining the full `lines` list on every budget check (O(n) per check → O(n²) total), the code maintains `_char_count` as a running sum and derives tokens from it in O(1) via `_current_tokens()`. The three closures (`_add_line`, `_current_tokens`, `_over_budget`) share this counter through `nonlocal`.

**Summary node collapsing.** IN nodes that have a `summarizes` metadata key absorb their covered nodes — covered IDs are hidden from output, and the summary node gets a `[summary]` prefix and a "(covers N nodes)" suffix. This prevents double-counting when a summary node abstracts over a cluster of related beliefs.

**Budget reservation for footer.** The footer line (`Token count: ~X / Y budget`) is pre-computed and reserved before any section starts filling. This guarantees the footer always fits.

## Dependencies

**Imports:**
- `date` from `datetime` — used only for the header timestamp
- `Network` from `.network` — the graph data structure; `compact` reads `network.nodes` (dict of `Node`) and `network.nogoods` (list of `Nogood`)

**Imported by:**
- `reasons_lib/api.py` — exposes `compact` as part of the functional API
- `tests/test_compact.py` and `tests/test_summarize.py` — test coverage

The module has no side effects and no I/O. It's a pure function over the in-memory network.

## Flow

1. **Partition** all nodes into `in_nodes` and `out_nodes` by truth value.
2. **Sort** `in_nodes` by dependent count (descending) — most-depended-on beliefs surface first.
3. **Emit header** with date, total counts, and nogood count.
4. **Section 1 — Nogoods**: Iterate `network.nogoods`, emitting each with its resolution if present. Stop when budget is hit.
5. **Section 2 — OUT nodes**: Iterate `out_nodes`, annotating each with its retraction/stale reason or supersession info from metadata. Stop when budget is hit.
6. **Section 3 — IN nodes**: Split into summary nodes and regular nodes. Build `covered_by_summary` set, then construct `visible_nodes` = summaries + non-covered regulars, re-sorted by dependent count. For each visible node, emit its text, dependency chain (from the first non-`summarizes` justification), dependent count, and summary coverage. Stop when budget is hit.
7. **Emit footer** with actual vs. budget token count.
8. **Join and return.**

## Invariants

- **Budget is respected**: every line addition checks `_over_budget` before appending. The footer is pre-reserved.
- **Priority ordering is strict**: nogoods → OUT → IN. A later section never displaces an earlier one.
- **Summary coverage is exclusive**: a node ID in `covered_by_summary` is hidden from the visible list even if it has many dependents. The summary node takes its place.
- **Token estimate is monotonically increasing**: `_char_count` only grows (lines are never removed), so `_current_tokens()` never decreases mid-execution.
- **At least 1 token**: `estimate_tokens` and `_current_tokens` both floor at 1.

## Error Handling

There is essentially none, by design. The function operates on an already-loaded `Network` object and performs no I/O. If the network is empty, the output is just the header and footer. If the budget is extremely small (e.g., less than the header itself), sections are simply skipped — the `_over_budget` checks gate every section header and every line. No exceptions are raised.

The one implicit assumption is that `node.metadata` is always a dict (never `None`). This is guaranteed by the `Node` dataclass in `__init__.py`.

## Topics to Explore

- [file] `tests/test_compact.py` — Exercises budget boundaries, truncation behavior, and the omission markers; shows expected output format
- [file] `tests/test_summarize.py` — Tests the summary node collapsing path where `metadata["summarizes"]` hides covered nodes
- [function] `reasons_lib/api.py:compact` — The API wrapper that loads from SQLite and calls this function; shows how callers invoke it
- [file] `reasons_lib/__init__.py` — The `Node`, `Justification`, and `Nogood` dataclasses that this module reads from; defines the `metadata` dict contract
- [general] `token-budget-accuracy` — The chars/4 heuristic diverges from real tokenizers on code, URLs, and non-Latin text; worth understanding where the budget over/under-counts

## Beliefs

- `compact-budget-guarantee` — `compact()` output estimated token count never exceeds the `budget` parameter; every line is gated by `_over_budget` before appending
- `compact-priority-order` — Sections are emitted in fixed order: nogoods, OUT nodes, IN nodes; a later section never displaces content from an earlier one
- `compact-pure-function` — `compact()` has no side effects, performs no I/O, and does not mutate the `Network` instance
- `compact-summary-hiding` — Nodes whose IDs appear in any summary node's `metadata["summarizes"]` list are excluded from the visible IN node output
- `compact-char-tracking-o1` — Budget tracking uses a running `_char_count` integer, making per-line budget checks O(1) instead of O(n)

