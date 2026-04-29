# File: tests/test_compact.py

**Date:** 2026-04-29
**Time:** 17:08



# `tests/test_compact.py` — Tests for the Belief Network Compact Summary

## Purpose

This file tests `reasons_lib.compact`, which produces a **token-budgeted, human-readable summary** of a belief network's state. The compact summary is designed for LLM context windows — it serializes the current truth-maintenance state (IN nodes, OUT nodes, nogoods, dependencies) into markdown that fits within a token budget. This is the mechanism behind the `BELIEF STATE` preservation described in the project's compaction instructions.

The file owns two responsibilities:
1. Verifying the `estimate_tokens` heuristic behaves correctly at boundaries.
2. Verifying that `compact()` produces structurally correct, budget-respecting, priority-ordered summaries.

## Key Components

### `TestEstimateTokens`

Tests the token estimation heuristic (`estimate_tokens`). Three boundary cases:

- **`test_uses_char_count_not_word_count`** — Confirms the heuristic is `len(text) // 4` (chars/4), *not* word count. This is important because the two diverge significantly for long words vs. short words.
- **`test_minimum_one_token`** — Empty string and very short strings floor to 1, never 0. Prevents division-related edge cases downstream.
- **`test_long_text`** — Sanity check: 400 chars → 100 tokens.

### `TestCompact`

Tests the `compact()` function across several dimensions:

| Test | What it verifies |
|------|-----------------|
| `test_empty_network` | Graceful handling of zero nodes — produces a valid summary with "0 nodes tracked" |
| `test_includes_nogoods` | Nogood sets (contradictions) appear under `## Nogoods` with auto-generated IDs |
| `test_includes_out_nodes` / `test_includes_in_nodes` | Retracted and active nodes appear in their respective sections |
| `test_truncates_long_text` / `test_no_truncate` | Long belief text is ellipsis-truncated when `truncate=True`, preserved when `False` |
| `test_budget_limits_*` (×3) | Each section (IN, OUT, nogoods) independently respects the token budget and emits "more … omitted" messages |
| `test_budget_respected_across_all_sections` | Combined budget enforcement: a mixed network with IN, OUT, and nogoods stays under budget (with a 25% tolerance for structural overhead) |
| `test_most_depended_on_first` | IN nodes are sorted by dependent count descending — the most "load-bearing" beliefs appear first, surviving truncation |
| `test_shows_dependencies` | Derived nodes show their antecedents (`<- a`) |
| `test_shows_dependent_count` | Nodes with dependents show the count (`(2 dependents)`) |
| `test_stale_reason_in_out` | OUT nodes with `stale_reason` metadata surface the reason (`stale: new data`) |
| `test_token_count_line` | The output includes a self-reporting token count line with the budget |

## Patterns

**Arrange-Act-Assert with in-memory networks.** Every test constructs a `Network()` from scratch, populates it with `add_node`/`retract`/`add_nogood`, calls `compact()`, and asserts on the resulting string. No database, no fixtures, no I/O.

**Budget as a first-class parameter.** The `compact()` function takes an explicit `budget` (in estimated tokens) and a `truncate` flag. Tests verify both the opt-in truncation of individual entries and the global budget cap that drops entire entries with "omitted" messages.

**Priority ordering tested via string position.** `test_most_depended_on_first` uses `result.index()` to compare positions of "root:" and "leaf:" in the output — a simple but effective way to assert ordering without parsing structured output.

## Dependencies

**Imports:**
- `reasons_lib.Justification` — dataclass/namedtuple for SL/CP justifications (type + antecedents + optional outlist)
- `reasons_lib.network.Network` — the in-memory belief graph (nodes, justifications, nogoods, truth values)
- `reasons_lib.compact.compact` — the function under test: serializes a Network to a budgeted markdown summary
- `reasons_lib.compact.estimate_tokens` — the `chars // 4` heuristic, tested independently

**Imported by:** Nothing — this is a leaf test file.

## Flow

1. A `Network` is constructed (empty graph).
2. Nodes are added with `add_node(id, text, justifications=[], metadata={})`.
3. Optional mutations: `retract()` flips nodes OUT, `add_nogood()` registers contradiction sets.
4. `compact(net, budget=N, truncate=bool)` walks the network:
   - Collects IN nodes, OUT nodes, and nogoods.
   - Sorts IN nodes by dependent count (descending).
   - Emits markdown sections, tracking cumulative token usage.
   - Truncates individual entries (if `truncate=True`) and drops remaining entries when the budget is exhausted.
   - Appends a self-reporting token-count footer.
5. Tests assert on substrings in the resulting markdown.

## Invariants

- **Token floor:** `estimate_tokens` never returns 0 — minimum is 1.
- **Budget soft-cap:** Output token count stays below `budget * 1.25` (the 25% tolerance in `test_budget_respected_across_all_sections`). Exact enforcement isn't possible because section headers and truncation messages themselves consume tokens.
- **Priority preservation:** When budget forces omission, high-dependent-count nodes survive over low-dependent-count nodes.
- **Section presence:** Even an empty network produces valid output with "0 nodes tracked".
- **Metadata surfacing:** If a node has `stale_reason` in its metadata and is OUT, the reason appears in the output.

## Error Handling

There is no explicit error handling in these tests — and that's intentional. The `compact()` function is designed to be infallible: it takes a `Network` (which is always in a valid state) and produces a string. Empty networks, zero-budget, and missing metadata are all handled gracefully by the implementation rather than raising exceptions. The tests verify this graceful degradation.

---

## Topics to Explore

- [file] `reasons_lib/compact.py` — The implementation under test: how the budget tracking, section rendering, and priority sorting actually work
- [file] `reasons_lib/network.py` — The `Network` class that `compact` serializes: node storage, truth value management, dependent tracking, nogood registration
- [function] `reasons_lib/compact.py:compact` — The main serialization function: how it walks the network, allocates budget across sections, and decides what to omit
- [general] `compact-context-integration` — How the compact output integrates with Claude's context compaction (the `BELIEF STATE` instructions in CLAUDE.md) to preserve belief state across LLM turns
- [file] `tests/conftest.py` — Shared fixtures and configuration for the test suite

## Beliefs

- `compact-estimate-tokens-chars-div-4` — `estimate_tokens` uses `len(text) // 4` with a floor of 1, not word count or any external tokenizer
- `compact-sorts-in-nodes-by-dependents-descending` — IN nodes are emitted in descending order of dependent count so the most load-bearing beliefs survive budget truncation
- `compact-budget-is-soft-cap` — The token budget is approximate; structural overhead (headers, truncation messages) can cause up to ~25% overshoot
- `compact-surfaces-stale-reason-metadata` — OUT nodes with `stale_reason` in their metadata include the reason string in the compact output
- `compact-is-infallible` — `compact()` handles empty networks, zero-budget, and missing metadata without raising exceptions

