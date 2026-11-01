---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/compact.py

# `reasons/compact.py` — Token-Budgeted Belief State Summary

## Purpose

This module owns the responsibility of **serializing a full `Network` (a defeasible reasoning state) into a compact, human/LLM-readable text block that fits within a hard token budget**. The primary use case is injecting the current belief state into a `CLAUDE.md` file or an LLM context window without blowing the window's limits.

It does *not* mutate the network. It reads node truth values, metadata, and nogood lists, then emits a prioritized, truncated Markdown summary.

## Key Components

### `estimate_tokens(text: str) -> int`
A one-liner using the **chars ÷ 4** BPE approximation (standard for GPT-family tokenizers). Returns `max(1, ...)` so empty strings still cost a token. This is the only tokenization strategy in the module—no real tokenizer is loaded.

### `compact(network: Network, budget: int = 500, truncate: bool = True) -> str`
The sole public API. Contract:

| Param | Meaning |
|---|---|
| `network` | A `reasons.network.Network` with `.nodes` (dict of node-id → node), `.nogoods` (list), and node attributes: `.truth_value`, `.text`, `.dependents`, `.justifications`, `.metadata` |
| `budget` | Max token count for the entire output (default 500 ≈ 2000 chars) |
| `truncate` | If `True`, node text longer than 80 chars is cut to 77 + `"..."` |

Returns a single `str` (Markdown-flavored) that **always** fits within `budget` tokens (measured by the same `chars/4` heuristic).

### Internal helpers (closures inside `compact`)

- **`_text(node)`** — applies the 80-char truncation.
- **`_add_line(line)`** — appends to the `lines` list *and* increments the running `_char_count`. This is the only mutation point for the output buffer.
- **`_current_tokens()`** — `max(1, _char_count // 4)`. O(1) because `_char_count` is accumulated incrementally rather than re-joining the list.
- **`_over_budget(line)`** — predicts whether adding `line` *plus* the mandatory footer would exceed `budget`. This is the gate before every `_add_line` call.

## Patterns

1. **Priority-tiered budget allocation.** The code walks three sections in a fixed order (Nogoods → OUT → IN). Each section *greedily* consumes budget until it can't fit the next line, then emits a `"... (N more omitted)"` sentinel and moves to the next tier. Earlier tiers never yield space to later tiers.

2. **O(1) budget tracking via closure state.** `_char_count` is a `nonlocal` integer updated on every `_add_line`. This avoids the O(n) cost of `" ".join(lines)` on every budget check. The footer token cost is subtracted upfront so the check is conservative.

3. **Summary-aware deduplication (IN nodes only).** Nodes whose `metadata["summarizes"]` lists other node IDs are treated as "representatives." The covered nodes are hidden. This mirrors a "roll-up" pattern: one summary node stands in for a cluster of fine-grained nodes.

4. **Defensive metadata access.** Every metadata lookup uses `.get(...)` with fallbacks (`retract_reason` falls back to `stale_reason`, `superseded_by` is optional). No `KeyError` can escape.

5. **Mirror module.** `reasons_lib/compact.py` exists as a parallel copy (the repo has `reasons/` and `reasons_lib/` trees). This suggests a library-vs-CLI split or a transitional refactor.

## Dependencies

**Imports (this file → others):**
- `datetime.date` — for the summary header date stamp.
- `reasons.network.Network` — the data structure being serialized. Only `.nodes`, `.nogoods`, and node attributes are accessed.

**Imported by (others → this file):**
- `reasons/api.py` — the public API layer that exposes `compact` to external callers.
- `reasons_lib/compact.py` — the mirror copy (may re-export or delegate).
- `tests/test_compact.py` — direct unit tests.
- `tests/test_summarize.py` — tests the summary/dedup logic.

## Flow

```
compact(network, budget, truncate)
│
├─ 1. Partition network.nodes into in_nodes / out_nodes by truth_value
├─ 2. Sort in_nodes by len(dependents) descending
├─ 3. Compute header lines (date, counts)
├─ 4. Initialize _char_count, footer_tokens, closure helpers
│
├─ 5. SECTION: Nogoods
│     ├─ Gate: skip entirely if header already nears budget
│     ├─ For each nogood: build line, check _over_budget, add or break
│     └─ Append blank line
│
├─ 6. SECTION: OUT nodes
│     ├─ Gate: skip if budget exhausted
│     ├─ For each OUT node: build line (with retract/supersede reason), check budget
│     └─ Append blank line
│
├─ 7. SECTION: IN nodes
│     ├─ Gate: skip if budget exhausted
│     ├─ Split into summary_nodes vs regular_nodes (via metadata["summarizes"])
│     ├─ Remove regular nodes covered by any summary
│     ├─ Re-sort visible_nodes by dependent count
│     ├─ For each visible node: build line (prefix, deps, dep_count, sum_info)
│     └─ Append "N hidden by summaries" note if applicable
│
└─ 8. Append footer: "Token count: ~X / budget budget"
     Return "\n".join(lines)
```

Each section is **independent**: if the budget is exhausted before a section starts, that section is silently skipped (no empty header is emitted). Within a section, lines are added one at a time with a budget check, so you get a *prefix* of the section's content, never a partial line.

## Invariants

- **Budget is a hard ceiling.** The returned string's `chars/4` will never exceed `budget`. The `_over_budget` check always reserves room for the footer.
- **Priority is strict.** Nogoods are never omitted in favor of IN nodes. If budget can't fit a nogood, it's cut; IN nodes are not promoted.
- **IN nodes are ordered by `len(dependents)` descending.** The most structurally important nodes appear first.
- **Summary nodes subsume their targets.** If node A has `metadata["summarizes"] = ["b", "c"]`, then b and c are excluded from the IN listing. A summary node is always listed before the regular nodes it covers (it appears in `summary_nodes`, which is prepended).
- **Every node appears at most once** in the output.
- **The footer always states the actual token count** (as estimated by the same heuristic), not the budget.
- **Truncation is 80 chars max** (77 + `"..."`) and only applies when `truncate=True`.

## Error Handling

This module is notably **error-averse by design**:

- **No exceptions are raised** for normal operation. The function always returns a string.
- `estimate_tokens` guards against zero-length input with `max(1, ...)`.
- All `metadata` accesses use `.get()` with `None` defaults. A missing key results in an empty suffix, not a `KeyError`.
- There is **no validation** that `network` is well-formed (e.g., that `node.truth_value` is one of `"IN"`, `"OUT"`). An unexpected value would fall into the `out_nodes` bucket (the `else` branch).
- There is **no logging, no warnings**, and no try/except. If `network.nodes` or `network.nogoods` is missing, you get an `AttributeError` that propagates to the caller.
- The budget check is purely arithmetic—there's no rounding or overflow concern for realistic budgets.

The implicit contract: *if the `Network` object is valid per its own invariants, `compact()` will always return a non-empty string within budget.*

---

## Topics to Explore

- [file] `reasons/network.py` — Defines the `Network`, node, and nogood classes that `compact()` serializes; understanding `.dependents`, `.justifications`, and `.metadata` schemas is essential for extending the summary.
- [file] `tests/test_compact.py` — Exercises budget edge cases, priority ordering, and the "N more omitted" sentinel; shows exactly which invariants are contract-tested.
- [file] `tests/test_summarize.py` — Tests the summary/dedup logic (`metadata["summarizes"]`); reveals the expected shape of the `summarizes` list and how hidden nodes are counted.
- [file] `reasons/api.py` — The public surface that wraps `compact()`; shows how `budget` is set (hardcoded? config-driven?) and how the summary is injected into `CLAUDE.md`.
- [general] `reasons-vs-reasons-lib-split` — The repo has parallel `reasons/` and `reasons_lib/` trees with identical file names. Understanding whether one is a generated artifact, a vendored copy, or a transitional refactor affects where to make changes.

## Beliefs

- `compact-respects-budget-as-hard-ceiling` — The returned string's token count (measured by `len(text) // 4`) will never exceed the `budget` parameter, because every `_add_line` call is gated by `_over_budget` which reserves footer tokens.
- `compact-prioritizes-nogoods-over-in-nodes` — If the budget can fit a nogood line but not an IN node line, the nogood is included and the IN node is omitted; the three sections are evaluated in fixed order with no cross-tier reallocation.
- `compact-hides-nodes-covered-by-summaries` — Any IN node whose ID appears in another node's `metadata["summarizes"]` list is excluded from the output, and the covering node is labeled `[summary]` with a `(covers N nodes)` annotation.
- `compact-uses-chars-4-tokenization` — All budget arithmetic uses the `len(text) // 4` heuristic; no real tokenizer (tiktoken, etc.) is imported or used anywhere in the module.
- `compact-reads-network-immutably` — `compact()` never calls any setter, mutation method, or `__setitem__` on `network` or its nodes; it is a pure read-and-serialize function.
