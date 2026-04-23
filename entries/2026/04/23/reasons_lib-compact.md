# File: reasons_lib/compact.py

**Date:** 2026-04-23
**Time:** 16:42

## `reasons_lib/compact.py` — Token-Budgeted Belief State Summary

### 1. Purpose

This file is the **serialization layer for LLM context injection**. It takes a `Network` (the in-memory TMS/reason maintenance system) and produces a Markdown-formatted summary that fits within a token budget. The output is designed to be pasted into `CLAUDE.md` files or injected into LLM context windows so that an agent can quickly understand which beliefs are held, which are retracted, and where contradictions exist.

It owns one responsibility: triage a potentially large belief network down to the most decision-relevant information within a fixed space budget.

### 2. Key Components

**`estimate_tokens(text: str) -> int`** (line 13)
A deliberately rough heuristic — splits on whitespace and counts words. Not a real tokenizer, but good enough for budgeting since the output is mostly short English phrases and node IDs. The budget parameter throughout the module is measured in this unit (word count), not BPE tokens.

**`compact(network: Network, budget: int = 500, truncate: bool = True) -> str`** (line 18)
The sole public function. Takes a Network and returns a Markdown string. The three parameters control:
- `network`: the TMS to summarize
- `budget`: maximum word count (default 500)
- `truncate`: whether to clip individual node text at 80 characters

The return value is a complete Markdown document with a header, statistics line, and up to three sections.

### 3. Patterns

**Priority-based triage.** The function implements a strict information priority:

1. **Nogoods** — always included, never budget-limited (lines 65–70). These are contradictions and are the most actionable items in any reasoning context.
2. **OUT nodes** — always included, never budget-limited (lines 73–83). Retracted beliefs with their retraction reasons.
3. **IN nodes** — budget-limited, ordered by dependent count (lines 87–142). These fill remaining space.

This is a deliberate design choice: the budget only constrains the least critical section. Nogoods and OUT nodes are unconditionally rendered, which means the actual output can exceed `budget` if there are many contradictions or retractions.

**Summary-based compression** (lines 89–107). IN nodes that are "covered" by a summary node (metadata key `summarizes`) are hidden from the output, replaced by their summary. This is the compact module's integration point with `Network.summarize()` — summaries are a first-class mechanism for reducing token usage. Covered nodes are only hidden when the summary itself is IN; if the summary is OUT (because one of its antecedents was retracted), the covered nodes reappear.

**Greedy packing** (lines 115–138). IN nodes are emitted one at a time, checking the running token count against the budget. When the budget is exceeded, a "... (N more IN nodes omitted)" line is appended and emission stops.

### 4. Dependencies

**Imports:**
- `datetime.date` — for the summary header timestamp
- `.network.Network` — the core data structure. `compact` reads `network.nodes` (dict of `Node`) and `network.nogoods` (list of `Nogood`). It never mutates the network.

**Imported by:**
- `reasons_lib/api.py` — the public API wraps `compact()` at line 1058, loading the network from storage first
- `tests/test_compact.py` — unit tests for the function itself
- `tests/test_summarize.py` — integration tests verifying that summaries affect compact output

### 5. Flow

1. **Partition** nodes into `in_nodes` and `out_nodes` by truth value (lines 35–41).
2. **Sort** IN nodes by `len(n.dependents)` descending — most-depended-on first (line 44).
3. **Emit header** with date, counts (lines 52–56).
4. **Emit nogoods section** — each nogood shows its ID, involved node IDs, and optional resolution (lines 65–70).
5. **Emit OUT section** — each OUT node shows its text plus a reason annotation derived from metadata. The annotation logic checks `retract_reason` and `stale_reason` first, then falls back to `superseded_by` (lines 73–83).
6. **Emit IN section** with summary compression:
   - Separate IN nodes into summary nodes and regular nodes (lines 92–100).
   - Compute `covered_by_summary` — the set of node IDs that a summary replaces (lines 89, 97–98).
   - Build `visible_nodes` = summaries + uncovered regulars, re-sorted by dependent count (lines 103–107).
   - Greedily emit lines until budget is exhausted (lines 115–138).
   - Each IN line includes: optional `[summary]` prefix, node ID, truncated text, first non-summary justification's antecedents (`<- a, b`), dependent count, and summary coverage count.
7. **Emit footer** with actual token count vs. budget (line 145).

### 6. Invariants

- **Nogoods and OUT nodes are never dropped by the budget.** The budget check only applies to section 3 (IN nodes). This means output can exceed the budget if there are many nogoods or OUT nodes.
- **Summary hiding is truth-value-aware.** A node is only hidden if its covering summary is IN. If the summary is OUT (e.g., one of its covered nodes was retracted), the remaining IN nodes among the covered set become visible again. This is enforced by only iterating over `in_nodes` when building the summary coverage set — an OUT summary won't appear in `in_nodes`.
- **IN nodes are ordered by dependent count.** The most structurally important nodes (those that other nodes depend on) are emitted first and therefore survive budget truncation.
- **The function is read-only.** It never mutates the network — purely a view.
- **Truncation is character-based, not token-based.** The `truncate` flag clips node text at 80 characters (77 + "..."), while the budget is measured in words.

### 7. Error Handling

There is none, and none is needed. The function reads from an already-validated `Network` instance. It doesn't access storage, IO, or external services. If the network is empty, it produces a valid summary with "0 nodes tracked". The only implicit assumption is that `node.metadata` is a dict (guaranteed by `Network.add_node`).

---

## Topics to Explore

- [file] `reasons_lib/network.py` — The `Network` class that compact reads from; understanding `Node`, `Justification`, `Nogood`, and truth value propagation is essential to understanding what compact is summarizing
- [function] `reasons_lib/network.py:summarize` — Creates summary nodes with `metadata["summarizes"]`, which is the mechanism compact uses for compression
- [file] `reasons_lib/export_markdown.py` — Likely the full (non-budgeted) Markdown export; compare with compact to understand what gets lost during triage
- [function] `reasons_lib/api.py:compact` — The public API entry point that loads the network from SQLite and delegates to this function
- [general] `budget-overflow-behavior` — Nogoods and OUT nodes bypass the budget, so compact output can exceed the specified limit; worth verifying whether callers account for this

---

## Beliefs

- `compact-budget-only-limits-in-nodes` — The token budget only constrains the IN nodes section; nogoods and OUT nodes are always emitted regardless of budget, so output can exceed the budget value.
- `compact-is-read-only` — `compact()` never mutates the Network; it is a pure view function.
- `compact-summary-hiding-requires-in` — A summary node only hides its covered nodes when the summary itself is IN; if the summary is OUT, covered nodes reappear in the output.
- `compact-token-estimate-is-word-count` — `estimate_tokens` counts whitespace-separated words, not BPE tokens; the budget parameter is measured in this unit.
- `compact-in-nodes-ordered-by-dependents` — IN nodes are sorted by descending dependent count, so structurally important nodes survive budget truncation.

