---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_compact.py

# `tests/test_compact.py` — Tests for Compact Network Summary

## Purpose

This file is the unit test suite for `reasons/compact.py`, the module responsible for producing a **bounded, text-based snapshot** of a belief `Network`. In the `ftl-reasons` project, beliefs live in a graph of nodes (premises, derivations) linked by justifications, with some nodes retracted (OUT) and some marked as inconsistent (nogoods). The `compact` function serializes that entire state into a single string summary — the kind of thing you'd hand to an LLM as context, or display in a CLI. This test file owns verifying that serialization is **correct, budgeted, ordered, and well-formed**.

It does not test the network logic itself (that's `test_network.py`); it tests the *projection* of network state into a token-budgeted text summary.

## Key Components

### `TestEstimateTokens`

Tests the `estimate_tokens(text: str) -> int` helper from `reasons.compact`.

| Test | Contract verified |
|---|---|
| `test_uses_char_count_not_word_count` | `estimate_tokens(t) == len(t) // 4`. Explicitly asserts it is **not** word-based (`!= len(t.split())`). This is the core formula: a crude 4-chars-per-token heuristic. |
| `test_minimum_one_token` | The function has a **floor of 1**. Empty string `""` and short `"hi"` both return 1, not 0. Prevents zero-token entries in budget accounting. |
| `test_long_text` | Linear scaling: 400 chars → 100 tokens. Confirms the `// 4` is a simple integer division with no other transformations. |

### `TestCompact`

Tests `compact(net: Network, budget: int | None = None, truncate: bool = True) -> str`.

**Content & structure tests** (no budget):

- `test_empty_network` — An empty `Network()` must still produce a valid summary containing `"Belief State Summary"` and `"0 nodes tracked"`. Guarantees the function doesn't blow up on zero nodes.
- `test_includes_nogoods` — Nogoods get a `## Nogoods` section. The nogood ID is deterministically constructed as `"nogood-a-b"` (concatenation of member node IDs).
- `test_includes_out_nodes` / `test_includes_in_nodes` — Retracted nodes appear under `## OUT (retracted)`; active nodes under `## IN (active)`. Node lines are formatted as `"<id>: <text>"`.
- `test_truncates_long_text` / `test_no_truncate` — With `truncate=True` (default), long node text is clipped and an ellipsis `"..."` is appended. With `truncate=False`, the full 200-char string survives intact.
- `test_stale_reason_in_out` — If a retracted node carries `metadata={"stale_reason": "..."}`, the summary annotates it with `"stale: new data"`. This surfaces *why* a belief was dropped.

**Budget tests** (the core contract):

- `test_budget_limits_in_nodes` — 50 active nodes with `budget=100` triggers `"more IN nodes omitted"`. The function does **not** raise; it silently drops nodes and appends a notice.
- `test_budget_limits_out_nodes` — Same behavior for retracted nodes: `"more OUT nodes omitted"`.
- `test_budget_limits_nogoods` — Same for nogoods: `"more nogoods omitted"`.
- `test_budget_respected_across_all_sections` — The budget is **global**, not per-section. With a mixed network (1 IN, 1 nogood, 20 OUT, 20 IN) and `budget=200`, the actual token count stays under 250 (allowing ~25% structural overhead for headers and truncation messages). At least the IN section must show its omission notice.

**Ordering & relationship tests:**

- `test_most_depended_on_first` — Within the IN section, nodes are sorted by **dependent count descending**. A node with one dependent (`root`) appears before a node with zero (`leaf`). Verified by string-position comparison: `result.index("root:") < result.index("leaf:")`.
- `test_shows_dependencies` — If node `b` is derived from `a` via a `Justification(type="SL", antecedents=["a"])`, the summary includes `"<- a"` to show the antecedent link.
- `test_shows_dependent_count` — A node with two dependents displays `"(2 dependents)"`.

**Footer:**

- `test_token_count_line` — The output always ends with a line containing `"Token count:"` and `"/ 500 budget"` (reflecting the `budget` parameter passed in).

## Patterns

- **Isolated test construction**: Every test builds its own `Network()` from scratch. There are no shared fixtures or `setUp` methods, which keeps each test self-contained and order-independent.
- **String-assertion style**: The tests validate the *textual contract* of `compact()` via substring checks (`"## Nogoods" in result`, `"<- a" in result`). This is appropriate because `compact()` returns a plain `str`, not a structured object. The cost is brittleness if the formatting changes, but it directly encodes the expected output shape.
- **Negative assertions**: Several tests assert the *absence* of content (e.g., `"A" * 200 not in result` when truncated, `estimate_tokens != len(text.split())`). This guards against regressions where the function accidentally includes or uses the wrong metric.
- **Budget-as-soft-limit**: The `test_budget_respected_across_all_sections` test uses a 25% tolerance window (`< 250` for a 200 budget) with a comment explaining structural lines cause overshoot. This signals that `budget` is a *heuristic cap*, not a hard limit.
- **No mocking / no I/O**: All tests operate on in-memory `Network` objects. No LLM calls, no database, no file I/O. This is a pure-logic module, and the tests reflect that.

## Dependencies

**Imports (upstream):**

| Symbol | Source | Role |
|---|---|---|
| `Justification` | `reasons` (package `__init__`) | Constructs `SL`-type justifications to build small dependency graphs |
| `Network` | `reasons.network` | The belief graph being summarized |
| `compact` | `reasons.compact` | The function under test |
| `estimate_tokens` | `reasons.compact` | Helper under test |

**Imported-by (downstream):**

This is a test file. Nothing in production code imports it. It is discovered and executed by pytest via the `tests/` directory convention. The `tests/conftest.py` may inject shared fixtures (e.g., a temporary DB or LLM stub), but this file does not appear to use any — all tests are self-contained.

## Flow

1. **`TestEstimateTokens`** — Pure function calls. Input is a string; output is an `int`. No state, no side effects. The three tests form a small decision table: normal text, degenerate (empty/short) text, and long text.

2. **`TestCompact`** — Each test follows the same shape:
   ```
   net = Network()
   net.add_node(...)          # build state
   net.add_nogood(...)        # (optional)
   net.retract(...)           # (optional)
   result = compact(net, ...)  # call SUT
   assert "..." in result     # verify text contract
   ```
   The `compact` function internally:
   - Iterates the network's IN (active) nodes, sorted by dependent count descending.
   - Iterates OUT (retracted) nodes.
   - Iterates nogoods.
   - For each section, checks cumulative token cost against `budget`. Once budget is exhausted, stops adding nodes and appends a `"<N> more <section> omitted"` line.
   - Truncates individual node text if `truncate=True`.
   - Appends a footer with the total token count and the budget.

3. **Budget enforcement** is the most complex flow: the function must track a running token total *across all three sections* (IN, OUT, nogoods) and stop adding entries when the budget is hit, regardless of which section is currently being processed.

## Invariants

- `estimate_tokens` **always returns ≥ 1**. The `max(1, len(text) // 4)` pattern is a hard floor.
- Token estimation is **`len(text) // 4`**, not word-count, not byte-count, not a BPE tokenizer call. It's a deliberate, cheap heuristic.
- The compact output **always** contains the header `"Belief State Summary"` and a `"Token count:"` footer line, even for an empty network.
- Node ordering within IN is **dependent-count descending**. This is a soft preference (most-connected nodes first) so that the budget, when truncated, drops the least-important nodes.
- Budget is a **global cap across all sections**, not per-section. The test `test_budget_respected_across_all_sections` explicitly mixes all three sections and checks the total.
- Omission is **graceful**: the function appends a `"... omitted"` notice rather than raising or silently cutting mid-line.
- Nogood IDs follow the pattern `"nogood-" + "-".join(member_ids)`.

## Error Handling

This test file exercises **no error paths**. There are no `pytest.raises` blocks, no invalid-input tests, no exception assertions. The implicit assumption is:

- `compact()` is a **pure formatter**: it takes a valid `Network` and returns a `str`. It never raises.
- `estimate_tokens()` accepts any string (including empty) and returns a valid `int ≥ 1`. It never raises.
- Budget exhaustion is a **normal condition**, not an error. The function handles it by truncating and annotating, never by raising.

A developer extending this file would want to add tests for: very large networks (performance), unicode/multi-byte text (does `// 4` still make sense?), and `budget=0` or `budget=None` edge cases.

---

## Topics to Explore

- [file] `reasons/compact.py` — The implementation under test. Understanding how it iterates nodes, accumulates token cost, and formats sections will make every assertion in this file self-evident.
- [file] `reasons/network.py` — The `Network` class, especially `add_node`, `retract`, `add_nogood`, and how it tracks dependencies/dependents. The ordering invariant in `test_most_depended_on_first` depends on how `Network` computes dependent counts.
- [function] `reasons/compact.py:estimate_tokens` — The 4-chars-per-token heuristic. Worth checking whether other modules (e.g., `reasons/llm.py`, `reasons/api.py`) use the same formula or a different one, to ensure consistency across the codebase.
- [file] `tests/conftest.py` — Shared pytest fixtures. Even though this file doesn't appear to use any, knowing what `conftest.py` provides (e.g., a mock LLM, a temp DB) is essential before adding new test files that might need those fixtures.
- [general] `token-budget-strategy` — How the `budget` parameter interacts with LLM context windows in `reasons/api.py` or `reasons/ask.py`. The compact summary is likely the context payload for an LLM call, and the 4-chars-per-token heuristic directly affects whether the summary fits within model limits.

## Beliefs

- `estimate-tokens-floors-at-1` — `estimate_tokens` returns a minimum of 1 for any input, including the empty string, preventing zero-cost entries in budget accounting.
- `compact-budget-is-global-not-per-section` — The `budget` parameter in `compact()` constrains the total token count across all three sections (IN, OUT, nogoods) simultaneously, not independently per section.
- `compact-orders-in-nodes-by-dependent-count-desc` — Active (IN) nodes in the compact summary are ordered by descending dependent count, so budget truncation preferentially drops the least-connected nodes.
- `compact-never-raises-on-budget-exhaustion` — When the token budget is exhausted, `compact()` appends a "… omitted" notice and returns a valid string; it does not raise an exception or return `None`.
- `compact-includes-stale-reason-from-metadata` — Retracted nodes carrying `metadata["stale_reason"]` have that reason surfaced in the OUT section as `"stale: <reason>"`, providing provenance for why the belief was dropped.
