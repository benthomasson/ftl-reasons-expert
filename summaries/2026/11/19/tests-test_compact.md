---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_compact.py

## Purpose

`tests/test_compact.py` is the behavioral test suite for the compact belief-state summary renderer in `reasons.compact`. It verifies two public functions:

- `estimate_tokens(text) -> int`
- `compact(net, truncate=..., budget=...) -> str`

The file acts as a regression/specification test for the summary format used to compress a `Network` into a token-conscious text representation. It covers section headers, truncation, budgeting, dependency display, node ordering, nogood display, stale metadata, and token accounting.

It is a black-box test file: it constructs real `Network` objects, calls `compact`, and asserts on output fragments rather than mocking internals.

## Key Components

### Imports under test

- `from reasons import Justification` — constructs SL justifications with antecedents.
- `from reasons.network import Network` — the belief network being summarized.
- `from reasons.compact import compact, estimate_tokens` — the functions under test.

A repository-level caveat: the repo contains both `reasons/` and `reasons_lib/` trees, but this test imports only from `reasons`. It does not exercise `reasons_lib`.

### `TestEstimateTokens`

Tests the token-estimation heuristic:

- `test_uses_char_count_not_word_count`
  - Asserts `estimate_tokens(text) == len(text) // 4`.
  - Also asserts it is not equal to `len(text.split())`, guarding against word-count-based estimates.
- `test_minimum_one_token`
  - Empty string returns `1`.
  - `"hi"` returns `1`.
  - This implies the effective contract is `max(1, len(text) // 4)`.
- `test_long_text`
  - A 400-character string returns `100`.

### `TestCompact`

Tests the `compact(net, ...)` output contract. Each test creates a fresh `Network` and mutates it through public APIs:

- `net.add_node(id, text, justifications=..., metadata=...)`
- `net.retract(id)`
- `net.add_nogood([...])`

Important test cases:

- `test_empty_network`
  - Output contains `"Belief State Summary"`.
  - Output contains `"0 nodes tracked"`.

- `test_includes_nogoods`
  - Adds nodes `a`, `b` and nogood `["a", "b"]`.
  - Output contains `"## Nogoods"` and `"nogood-a-b"`.

- `test_includes_out_nodes`
  - Adds then retracts node `a`.
  - Output contains `"## OUT (retracted)"` and `"a: Premise A"`.

- `test_includes_in_nodes`
  - Adds active node `a`.
  - Output contains `"## IN (active)"` and `"a: Premise A"`.

- `test_truncates_long_text` and `test_no_truncate`
  - With `truncate=True`, a 200-character node text is shortened and the output contains `"..."`.
  - With `truncate=False`, the full `"A" * 200` string remains.

- Budget tests:
  - `test_budget_limits_in_nodes` expects `"more IN nodes omitted"`.
  - `test_budget_limits_out_nodes` expects `"more OUT nodes omitted"`.
  - `test_budget_limits_nogoods` expects `"more nogoods omitted"`.
  - `test_budget_respected_across_all_sections` accepts approximate budget behavior: with `budget=200`, output must be `< 250` tokens, and must include `"more IN nodes omitted"`.
  - `test_token_count_line` expects `"Token count:"` and `"/ 500 budget"`.

- Dependency behavior:
  - `test_most_depended_on_first`: `root` has one dependent (`dep`), `leaf` has zero. The output index of `"root:"` must be before `"leaf:"`.
  - `test_shows_dependencies`: derived node `b` with antecedent `a` causes output to contain `"<- a"`.
  - `test_shows_dependent_count`: node `a` is antecedent for both `b` and `c`, so output contains `"(2 dependents)"`.

- Metadata behavior:
  - `test_stale_reason_in_out`: retracted node with `metadata={"stale_reason": "new data"}` produces `"stale: new data"`.

## Patterns

- **Pytest class-based tests**: behavior grouped into `TestEstimateTokens` and `TestCompact`.
- **Arrange-Act-Assert**: each test constructs a `Network`, performs one mutation pattern, calls `compact` or `estimate_tokens`, then asserts on output.
- **Black-box contract testing**: tests assert on stable output fragments rather than exact full-string snapshots.
- **Isolation by construction**: every test creates its own `Network`; there are no shared fixtures or mutable global state.
- **Approximate budget testing**: `test_budget_respected_across_all_sections` explicitly tolerates minor overshoot because `estimate_tokens` is a char/4 heuristic and structural lines add overhead.
- **Public API only**: tests use `add_node`, `retract`, `add_nogood`, `Justification`, and `compact`, not private helpers.

## Dependencies

This file depends on:

- `reasons.Justification`
- `reasons.network.Network`
- `reasons.compact.compact`
- `reasons.compact.estimate_tokens`
- Pytest as the runner, although it is not imported directly.

Nothing in production code imports this test file. Its downstream dependency is the test runner/CI process. If `reasons.compact` changes output formatting, truncation behavior, section ordering, or budget semantics, these tests are the first likely failures.

## Flow

The general execution flow for most `TestCompact` tests is:

1. Create an empty `Network`.
2. Add nodes, retract nodes, attach justifications, attach metadata, or add nogoods.
3. Call `compact(net, ...)` with optional `truncate` and `budget` arguments.
4. Inspect the returned string with substring checks or index-order checks.

Examples:

- For truncation, a long node text is added, then `compact(net, truncate=True)` must abbreviate it and include `"..."`; `truncate=False` must preserve it.
- For dependency rendering, a derived node has a `Justification(type="SL", antecedents=["a"])`. The compact output must show that relationship as `"<- a"`.
- For ordering, `root` has one dependent and `leaf` has none. The compact output must place `"root:"` before `"leaf:"`.
- For budget overflow, many nodes are added until small budgets force omission messages such as `"more IN nodes omitted"`.

`estimate_tokens` is a pure function: string in, integer out. The tests establish that it uses character count, not word count, and never returns less than `1`.

## Invariants

The tests collectively enforce these rules:

- `estimate_tokens(text)` must be based on `len(text) // 4`.
- `estimate_tokens` must never return `0`; empty and very short strings return `1`.
- `compact` output must include `"Belief State Summary"`.
- Empty networks must report `"0 nodes tracked"`.
- The compact output must have section headings for relevant content:
  - `"## IN (active)"`
  - `"## OUT (retracted)"`
  - `"## Nogoods"`
- Node output uses an `id: text` style, e.g. `"a: Premise A"`.
- Retracted nodes appear in the OUT section.
- Nogood output includes an ID-like string such as `"nogood-a-b"`.
- `truncate=True` must shorten long node text and include `"..."`.
- `truncate=False` must preserve full node text.
- When budget pressure occurs, the output must include omission markers:
  - `"more IN nodes omitted"`
  - `"more OUT nodes omitted"`
  - `"more nogoods omitted"`
- Budget enforcement is approximate, not exact. A 200-token budget may produce up to 249 estimated tokens in the tested scenario.
- More-depended-on nodes should appear before less-depended-on nodes in the tested ordering case.
- Dependencies are shown as `"<- <antecedent-id>"`.
- Dependent count is shown as `"(N dependents)"`.
- Stale retracted nodes render `"stale: <reason>"`.
- The output includes a token-count line such as `"Token count: ... / 500 budget"`.

## Error Handling

This file does not test error paths. It does not use `pytest.raises`, try/except blocks, or invalid inputs. The implicit expectation is that `compact` handles empty networks, retracted nodes, many nodes, justifications, metadata, and nogoods without raising. Any exception from `compact` or `estimate_tokens` simply fails the test.

No validation is asserted for malformed network states, invalid node IDs, missing antecedents, `None` text, negative budgets, or unsupported truncation values.

## Topics to Explore

- [file] `reasons/compact.py` — Implementation under test; inspect section generation, truncation, and budget omission logic.
- [function] `reasons/compact.py:estimate_tokens` — Confirm the exact heuristic and any edge cases beyond the tested string lengths.
- [file] `reasons/network.py` — Understand `Network`, `add_node`, `retract`, `add_nogood`, and how dependent counts are derived.
- [general] `soft-budget-algorithm` — Explore why the budget is approximate and how structural lines affect token estimates.
- [file] `reasons_lib/compact.py` — Parallel package tree; compare with `reasons/compact.py` because this test imports only `reasons`.

## Beliefs

- `estimate-tokens-uses-char-count` — The tests require `estimate_tokens(text)` to equal `len(text) // 4` with a minimum return value of `1`.
- `compact-truncates-only-when-enabled` — `truncate=True` must insert `...` and suppress a 200-character node text; `truncate=False` must preserve it.
- `compact-budget-is-approximate` — A 200-token budget test allows output below 250 estimated tokens and expects omission markers for overflowed sections.
- `compact-orders-by-dependent-count` — In the tested scenario, a node with one dependent must appear before a node with zero dependents.
- `test-targets-reasons-not-reasons-lib` — This test imports from `reasons`, so the parallel `reasons_lib/compact.py` implementation is not exercised here.
