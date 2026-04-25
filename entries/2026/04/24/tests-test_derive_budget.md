# File: tests/test_derive_budget.py

**Date:** 2026-04-24
**Time:** 17:00

# `tests/test_derive_budget.py`

## Purpose

This is a regression test suite for **issue #23** — an O(n^2) budget calculation bug in `_build_beliefs_section`. The function builds a token-budgeted beliefs section for the LLM derive prompt, splitting a fixed budget between agent beliefs and local beliefs. The bug was that `count += len(belief_ids)` was placed *inside* the per-belief rendering loop rather than after it, causing `count` to accumulate `N * N` instead of `N`. This starved local beliefs via the formula `remaining = max(5, max_beliefs - count)`, which would collapse to the floor of 5 even with a generous budget.

The file's sole responsibility is ensuring the budget arithmetic in `_build_beliefs_section` and `build_prompt` is correct — linear, proportional, floored, and deterministic under sampling.

## Key Components

### Helpers (not under test)

- **`_make_nodes(agent_beliefs, local_beliefs)`** — Factory that builds a `nodes` dict matching `_build_beliefs_section`'s expected shape without touching SQLite. Deliberately omits `:active` premise nodes to avoid inflating agent counts (mirrors the function's `startswith` filtering).
- **`_count_local_shown(output)`** / **`_count_agent_shown(output, agent_name)`** — Regex extractors that parse the `(N beliefs, showing M)` header emitted by `_build_beliefs_section`. They assert the header exists and return `M` as an int. These are the primary assertion mechanism — the tests verify budget behavior through the output formatting, not internal state.

### Test Classes

| Class | Tests | What it covers |
|---|---|---|
| `TestCountLinearNotQuadratic` | 2 | Core regression — verifies `count` grows as N, not N^2 |
| `TestMultiAgent` | 2 | Budget accumulates correctly across 2–3 agents |
| `TestBudgetFloor` | 2 | Local beliefs get at least 5 even when agents consume the entire budget |
| `TestEdgeCases` | 6 | Boundary conditions: single belief, no agents, no locals, empty network, empty agents dict, derived beliefs |
| `TestSampling` | 2 | Sample mode respects budget and is deterministic with a fixed seed |
| `TestBuildPromptIntegration` | 1 | End-to-end through `build_prompt` with a real SQLite database |

## Patterns

**Output-based testing**: Rather than monkeypatching internals or checking intermediate variables, every test calls `_build_beliefs_section`, gets a string, and regex-parses the section headers. This makes the tests resilient to formatting changes in the belief lines themselves but coupled to the header format `(N beliefs, showing M)`.

**Database-free unit tests**: Most tests use `_make_nodes` to construct the data structures directly, avoiding SQLite overhead. Only `TestBuildPromptIntegration` creates a real database via the `db` fixture.

**Proportional budget formula under test** (from `derive.py:198`):
```
agent_budget = max(5, int(max_beliefs * len(agent_beliefs) / total_all))
```
followed by:
```
remaining = max(5, max_beliefs - count)
```
The docstrings in `test_six_beliefs_one_agent` and `test_floor_of_five_when_agents_exceed_budget` work through the formula numerically for both the fixed and buggy versions.

## Dependencies

**Imports**: `reasons_lib.api` (for the integration test's `init_db`, `add_node`, `export_network`), `reasons_lib.derive._build_beliefs_section` and `build_prompt` (functions under test).

**Imported by**: Nothing — this is a leaf test file.

## Flow

1. Construct a `nodes` dict with agent-namespaced keys (`agent-a:b0`) and/or local keys (`local-0`), plus empty `derived` and optionally an `agents` mapping.
2. Call `_build_beliefs_section(nodes, derived, agents, max_beliefs=N)`.
3. The function partitions beliefs into agent groups and local, allocates budget proportionally, truncates or samples, and returns a formatted string.
4. Tests parse the `showing N` count from the output headers and assert budget invariants.

For the integration test, step 1 is replaced by inserting nodes into SQLite via `api.add_node`, then extracting via `api.export_network`, and step 2 calls `build_prompt` which internally calls `_build_beliefs_section`.

## Invariants

1. **Linearity**: Total agent beliefs counted equals the sum of per-agent shown counts, not their product.
2. **Budget floor**: `local_shown >= 5` regardless of how many agent beliefs exist (`derive.py:218`).
3. **Budget ceiling**: `agent_shown + local_shown <= max_beliefs`.
4. **Proportionality**: `local_shown == max(5, max_beliefs - total_agent_shown)` for multi-agent cases.
5. **Determinism**: Same seed produces identical output in sample mode.
6. **No crash on empty**: Empty nodes, empty agents, or agents pointing to missing nodes all produce a valid string.

## Error Handling

Minimal — this is a test file. The helpers `_count_local_shown` and `_count_agent_shown` assert that the regex matches and fail with a diagnostic message showing the first 500 characters of output if the expected header format is missing. No exceptions are caught or suppressed; failures propagate as `AssertionError` through pytest.

## Topics to Explore

- [function] `reasons_lib/derive.py:_build_beliefs_section` — The function under test; trace the proportional budget allocation and the fixed `count +=` placement
- [file] `reasons_lib/derive.py` — Full derive module including prompt construction, proposal parsing, and validation
- [file] `tests/test_derive.py` — Companion test file covering `parse_proposals`, `validate_proposals`, and other derive functionality
- [function] `reasons_lib/derive.py:_sample_beliefs` — Reservoir sampling used in sample mode; relevant to the `TestSampling` class
- [general] `issue-23-n-squared-budget` — The original bug report and fix commit for the O(n^2) budget tracking

## Beliefs

- `budget-floor-is-five` — `_build_beliefs_section` guarantees local beliefs get at least 5 slots regardless of agent count, enforced by `max(5, max_beliefs - count)`
- `count-accumulates-linearly` — After the bug fix, `count` in `_build_beliefs_section` increments by `len(belief_ids)` once per agent, not once per rendered belief line
- `tests-verify-via-output-headers` — All budget assertions parse the `(N beliefs, showing M)` header string rather than inspecting internal variables
- `make-nodes-omits-active-premises` — The `_make_nodes` helper deliberately excludes `:active` nodes to match `_detect_agents`'s skip logic and avoid inflating agent counts
- `sample-mode-is-deterministic` — `_build_beliefs_section` with `sample=True` and a fixed `seed` produces identical output across calls

