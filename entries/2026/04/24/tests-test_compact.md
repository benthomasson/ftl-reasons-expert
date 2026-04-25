# File: tests/test_compact.py

**Date:** 2026-04-24
**Time:** 17:02



# `tests/test_compact.py` — Tests for Token-Budgeted Belief Network Summaries

## 1. Purpose

This file tests the `compact` module, which produces a text summary of a belief network under a token budget constraint. The compact feature exists to serialize the current state of a TMS network into a human-readable (and LLM-readable) markdown summary that fits within a token limit — critical for feeding belief state into context windows during conversation compaction.

The file owns two responsibilities:
- Verifying the `estimate_tokens` heuristic behaves correctly at boundaries
- Verifying the `compact` function produces well-structured, budget-respecting summaries with the right content and ordering

## 2. Key Components

### `TestEstimateTokens`

Tests the token estimation heuristic. Three cases cover the contract:
- **chars ÷ 4** is the formula (not word count) — `test_uses_char_count_not_word_count`
- **Minimum of 1** — empty and very short strings never return 0 — `test_minimum_one_token`
- **Linear scaling** — 400 chars → 100 tokens — `test_long_text`

### `TestCompact`

Tests the `compact()` function across several dimensions:

| Test | What it verifies |
|---|---|
| `test_empty_network` | Graceful handling of zero nodes; outputs "0 nodes tracked" |
| `test_includes_nogoods` | Nogood (contradiction) records appear in a `## Nogoods` section |
| `test_includes_out_nodes` / `test_includes_in_nodes` | Retracted and active nodes appear in separate sections |
| `test_truncates_long_text` / `test_no_truncate` | Long node text is clipped with `...` when `truncate=True` |
| `test_budget_limits_*` | Budget enforcement for IN nodes, OUT nodes, and nogoods individually |
| `test_budget_respected_across_all_sections` | End-to-end budget check — total output stays within ~125% of budget |
| `test_most_depended_on_first` | Nodes with more dependents sort earlier (triage priority) |
| `test_shows_dependencies` | Derived nodes show their antecedents (`<- a`) |
| `test_shows_dependent_count` | Nodes display how many other nodes depend on them |
| `test_stale_reason_in_out` | Stale metadata surfaces in the OUT section |
| `test_token_count_line` | Output includes a self-reporting token count vs. budget |

## 3. Patterns

- **Arrange-Act-Assert**: Every test creates a `Network`, populates it, calls `compact()` or `estimate_tokens()`, then asserts on the string output. No shared fixtures — each test is self-contained.
- **String containment checks**: The tests verify output structure by checking for section headers (`"## IN (active)"`), formatted entries (`"a: Premise A"`), and metadata markers (`"stale: new data"`). This is appropriate for a text-rendering function where exact format may evolve but key content must be present.
- **Positional ordering assertions**: `test_most_depended_on_first` uses `str.index()` to verify relative ordering — a clean idiom for testing sort behavior in rendered output.
- **Budget tolerance**: `test_budget_respected_across_all_sections` allows up to 25% overshoot (`actual < 250` for budget 200), acknowledging the chars÷4 heuristic and structural overhead (headers, truncation messages).

## 4. Dependencies

**Imports:**
- `reasons_lib.Justification` — dataclass for SL justifications (antecedent lists)
- `reasons_lib.network.Network` — the core dependency graph; used to build test scenarios
- `reasons_lib.compact.compact` — the function under test; renders a network to a budgeted string
- `reasons_lib.compact.estimate_tokens` — the token estimation heuristic under test

**Imported by:** Nothing — this is a leaf test module.

## 5. Flow

The typical test flow:

1. **Build a network** — `Network()` creates an empty graph, then `add_node()` / `retract()` / `add_nogood()` populate it
2. **Call `compact(net, budget=N, truncate=bool)`** — this walks the network, sorts nodes by dependency count, renders sections (IN, OUT, Nogoods), and truncates to fit the budget
3. **Assert on the rendered string** — presence of section headers, node entries, metadata annotations, and budget compliance

For `estimate_tokens`, the flow is simply: pass a string, verify the integer result.

## 6. Invariants

- **Token estimate ≥ 1**: `estimate_tokens` never returns 0, even for empty strings.
- **Section presence**: IN nodes, OUT nodes, and nogoods each get their own markdown section when present.
- **Budget is soft**: The output may slightly exceed the budget due to structural overhead, but `test_budget_respected_across_all_sections` enforces a ceiling of 125% of budget.
- **Dependency-count ordering**: Nodes with more dependents appear before nodes with fewer, ensuring the most structurally important beliefs are preserved when the budget forces truncation.
- **Truncation markers**: When nodes are omitted, the output includes `"more IN nodes omitted"`, `"more OUT nodes omitted"`, or `"more nogoods omitted"`.

## 7. Error Handling

There is no explicit error handling in the tests — and that's intentional. The tests verify happy-path behavior and edge cases (empty network, zero-length strings). The `compact` function itself is expected to handle all network states gracefully without raising exceptions; these tests implicitly verify that contract by not catching any exceptions.

---

## Topics to Explore

- [file] `reasons_lib/compact.py` — The implementation being tested; see how budget allocation is split across sections and how dependency sorting works
- [function] `reasons_lib/compact.py:estimate_tokens` — The chars÷4 heuristic and its minimum-1 floor; understand why this was chosen over tiktoken
- [function] `reasons_lib/network.py:add_nogood` — How contradictions are recorded and assigned IDs like `nogood-001`
- [general] `budget-allocation-strategy` — How the compact function divides a token budget across IN, OUT, and nogood sections — the tests reveal that truncation applies per-section, which implies some allocation policy
- [file] `reasons_lib/network.py` — The `Network` class is the core data structure; understanding `add_node`, `retract`, and dependency tracking is prerequisite to understanding what compact summarizes

## Beliefs

- `estimate-tokens-chars-div-4` — `estimate_tokens` uses `len(text) // 4` with a minimum return value of 1, never 0
- `compact-budget-soft-ceiling` — The compact function treats the token budget as approximate; structural overhead (headers, truncation messages) can cause output to exceed the budget by up to ~25%
- `compact-sorts-by-dependents` — IN nodes are rendered in descending order of dependent count, so the most structurally important beliefs survive budget truncation
- `compact-three-sections` — The compact output is organized into three markdown sections: `## IN (active)`, `## OUT (retracted)`, and `## Nogoods`
- `compact-self-reports-tokens` — When given a budget, the compact output includes a `Token count: N / B budget` line for auditability

