# File: tests/test_derive_budget.py

**Date:** 2026-05-08
**Time:** 14:18



# `tests/test_derive_budget.py` — Regression Tests for Belief Budget Calculation

## Purpose

This file is a targeted regression test suite for **issue #23**: a quadratic-counting bug in `_build_beliefs_section` within the derive module. The bug caused `count += len(belief_ids)` to execute inside the per-belief loop (once per belief per agent), inflating the count to N² instead of N. The practical consequence was that local beliefs got starved to the floor budget of 5, because `remaining = max(5, max_beliefs - count)` saw a massively inflated `count`.

The file tests the fix and guards the budget allocation invariants for the prompt-building pipeline — the part of `reasons_lib.derive` that decides how many agent beliefs vs. local beliefs to include when constructing an LLM derivation prompt under a token budget constraint.

## Key Components

### Helper Functions

- **`_make_nodes(agent_beliefs, local_beliefs)`** — Test data factory. Builds a `(nodes, derived, agents)` tuple without touching the database. Deliberately omits `:active` premise nodes to avoid inflating counts (since `_build_beliefs_section` matches agent beliefs by `startswith`). This is a design choice documented in the docstring.

- **`_count_local_shown(output)`** / **`_count_agent_shown(output, agent_name)`** — Output parsers that extract the "showing N" count from section headers using regexes. These couple the tests to the exact header format of `_build_beliefs_section`, e.g., `Local beliefs (26 beliefs, showing 15)`.

### Test Classes

| Class | What it tests |
|---|---|
| `TestCountLinearNotQuadratic` | Core regression — count accumulates N, not N² |
| `TestMultiAgent` | Budget sums correctly across 2–3 agents |
| `TestBudgetFloor` | Local beliefs always get at least 5, even when agents consume the entire budget |
| `TestEdgeCases` | N=1, no agents, no locals, empty network, derived-belief exclusion |
| `TestSampling` | Sample mode respects budget and is deterministic with a fixed seed |
| `TestBuildPromptIntegration` | End-to-end test through `build_prompt` with a real SQLite database |

### The `db` Fixture

A single `@pytest.fixture` that creates a temporary SQLite database via `api.init_db`. Only used by `TestBuildPromptIntegration` — every other test uses `_make_nodes` to avoid I/O.

## Patterns

1. **Arithmetic assertions, not string matching** — Tests verify budget invariants numerically (`local_shown == 15`, `total_used <= 20`, `local_shown >= 5`) rather than comparing full output strings. This makes them resilient to formatting changes.

2. **Regex-based output parsing** — `_count_local_shown` and `_count_agent_shown` parse structured headers from `_build_beliefs_section` output. This is a fragile coupling point, but it's the right tradeoff: the header format is part of the function's contract for LLM prompt readability.

3. **Unit vs. integration separation** — Most tests operate on in-memory dicts via `_make_nodes` (fast, no I/O). Only `TestBuildPromptIntegration` hits the database through `api.add_node` / `api.export_network` / `build_prompt`.

4. **Docstring-as-proof** — Several test docstrings include the exact arithmetic showing the expected vs. buggy values (e.g., "Proportional budget: agent_budget=max(5, int(20*6/26))=5, count=5, remaining=max(5, 20-5)=15. With the old bug: count=5*5=25, remaining=max(5, 20-25)=5."). This makes the regression immediately understandable without reading the original issue.

## Dependencies

**Imports:**
- `reasons_lib.api` — `init_db`, `add_node`, `export_network` (used only in integration test)
- `reasons_lib.derive` — `_build_beliefs_section` (the function under test), `build_prompt` (integration)
- `re`, `pytest` — standard

**Imported by:** Nothing — this is a leaf test module.

## Flow

The typical test flow:

1. Construct a synthetic belief network via `_make_nodes` with controlled agent/local counts.
2. Call `_build_beliefs_section(nodes, derived, agents, max_beliefs=N)`.
3. Parse the output string with `_count_local_shown` / `_count_agent_shown`.
4. Assert budget arithmetic: `local_shown == max(5, max_beliefs - agent_total)`.

The integration test adds front matter: it writes nodes into a real SQLite DB, exports the network as a dict, and calls `build_prompt` which internally calls `_build_beliefs_section`.

## Invariants

The tests collectively enforce these invariants of `_build_beliefs_section`:

1. **Linear counting**: After processing N agent beliefs, the accumulated count equals N, not N².
2. **Budget floor**: Local beliefs always get at least 5 slots, regardless of agent consumption.
3. **Budget conservation**: `agent_shown + local_shown <= max_beliefs`.
4. **Multi-agent summation**: Count accumulates across all agents, not per-agent.
5. **Deterministic sampling**: Same seed produces identical output.
6. **Graceful degradation**: Empty networks, missing agents, and zero-local scenarios don't crash.

## Error Handling

The tests themselves don't test error paths in the production code — they verify correctness of the budget math. The helper `_count_local_shown` and `_count_agent_shown` fail with descriptive `AssertionError` messages (including the first 500 chars of output) when the expected header format isn't found, which makes debugging format changes straightforward.

---

## Topics to Explore

- [function] `reasons_lib/derive.py:_build_beliefs_section` — The function under test; read it to see the proportional budget formula and the fixed counting logic
- [function] `reasons_lib/derive.py:build_prompt` — The public entry point that calls `_build_beliefs_section`; understanding its `budget` parameter and `stats` return value contextualizes the integration test
- [file] `reasons_lib/api.py` — Provides `add_node`, `export_network`, and `init_db` used in the integration test; understanding the network export format clarifies what `nodes`, `derived`, and `agents` dicts actually look like in production
- [file] `tests/test_derive.py` — The sibling test file for the derive module, likely covering prompt construction and derivation logic beyond budget math
- [general] `agent-belief-prefix-convention` — The `startswith` matching used to identify agent beliefs (e.g., `agent-a:belief-1`) is a convention that affects budget counting and is why `_make_nodes` deliberately omits `:active` nodes

## Beliefs

- `derive-budget-count-is-linear` — `_build_beliefs_section` accumulates agent belief count as N (number of beliefs shown), not N² (beliefs × belief_ids list length), as verified by `TestCountLinearNotQuadratic`
- `derive-budget-local-floor-is-five` — Local beliefs always receive at least 5 slots in the budget via `max(5, max_beliefs - count)`, even when agent beliefs exceed the total budget
- `derive-budget-sample-is-deterministic` — `_build_beliefs_section` with `sample=True` and a fixed `seed` produces identical output across calls
- `derive-budget-tests-parse-header-format` — The test helpers `_count_local_shown` and `_count_agent_shown` depend on the exact header format `(N beliefs, showing M)` — changing that format in `_build_beliefs_section` will break these tests

