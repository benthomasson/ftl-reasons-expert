# File: tests/test_derive_budget.py

**Date:** 2026-04-29
**Time:** 17:03

# `tests/test_derive_budget.py`

## Purpose

This file is a regression test suite for **issue #23** — a quadratic counting bug in `_build_beliefs_section` within the derive module. The bug caused `count += len(belief_ids)` to execute inside a per-belief loop, inflating the count from N to N², which starved the local beliefs budget via `remaining = max(5, max_beliefs - count)`. The file's sole responsibility is ensuring the budget arithmetic in `_build_beliefs_section` and `build_prompt` is correct across all configurations of agents, local beliefs, and edge cases.

## Key Components

### Helpers

- **`_make_nodes(agent_beliefs, local_beliefs)`** — Factory that builds the `(nodes, derived, agents)` triple expected by `_build_beliefs_section`, without touching a database. Agent belief IDs follow the `agent:suffix` convention. Critically, it does *not* create `:active` premise nodes — the docstring explains this would inflate agent counts since the function under test matches by `startswith`.

- **`_count_local_shown(output)`** / **`_count_agent_shown(output, agent_name)`** — Regex extractors that parse the formatted output string for `"showing N"` counts in section headers. They assert the header exists before returning, so a missing header is a test failure, not a silent `None`.

### Test Classes

| Class | What it covers |
|---|---|
| `TestCountLinearNotQuadratic` | Core regression: verifies count accumulates as N, not N². Two tests — one checks the local budget directly, the other checks total usage stays within `max_beliefs`. |
| `TestMultiAgent` | Verifies that budget accumulation sums correctly across 2 and 3 agents, and that the local remainder formula `max(5, budget - agent_total)` holds. |
| `TestBudgetFloor` | Ensures the `max(5, ...)` floor guarantees at least 5 local beliefs are shown, even when agents consume all or most of the budget. |
| `TestEdgeCases` | Boundary conditions: single agent belief (where N²=N, so the bug was invisible), no agents, no local beliefs, empty network, agents with no matching nodes, and derived beliefs with justifications. |
| `TestSampling` | Verifies the budget fix also applies in sample mode (`sample=True`), and that deterministic seeding produces identical output. |
| `TestBuildPromptIntegration` | End-to-end test that goes through `build_prompt` with a real SQLite database, creating nodes via `api.add_node` and exporting via `api.export_network`. Verifies the budget is respected in the full pipeline. |

### Fixture

- **`db(tmp_path)`** — Creates a throwaway SQLite database via `api.init_db`. Only used by `TestBuildPromptIntegration`.

## Patterns

1. **Black-box output parsing** — Tests treat `_build_beliefs_section` as a string-producing function and parse its output with regex. This decouples tests from internal data structures but couples them to the output format (specifically the `"showing N"` header pattern).

2. **Algebraic assertions** — Rather than hardcoding expected values, most assertions express the budget invariant algebraically: `local_shown == max(5, budget - agent_total)`. This makes the tests self-documenting and resilient to minor changes.

3. **Bug-invisible edge case** — `test_single_agent_belief` explicitly tests N=1 where N²=N=1, noting the bug was invisible at this size. This is a good practice for regression suites — cover the case where the bug *didn't* manifest to ensure the fix doesn't break it.

4. **No `:active` nodes in helpers** — `_make_nodes` deliberately avoids creating `:active` premise nodes because `_build_beliefs_section` matches agent beliefs via `startswith(agent_name + ":")`, and an `:active` node would be counted as a regular agent belief.

## Dependencies

**Imports:**
- `re` — for parsing output headers
- `pytest` — test framework, `tmp_path` fixture
- `reasons_lib.api` — `init_db`, `add_node`, `export_network` (used only in integration test)
- `reasons_lib.derive` — `_build_beliefs_section` (the function under test), `build_prompt` (integration test)

**Imported by:** Nothing — this is a leaf test module.

## Flow

1. Each test constructs a `(nodes, derived, agents)` triple — either via `_make_nodes` (unit tests) or via `api.add_node` + `api.export_network` (integration test).
2. Calls `_build_beliefs_section(nodes, derived, agents, max_beliefs=N)` to get a formatted string.
3. Parses the string with `_count_local_shown` / `_count_agent_shown` to extract the actual "showing" counts.
4. Asserts the counts satisfy the budget invariant.

The integration test adds a database layer: it creates nodes with SL justifications pointing at `:active` premises (mimicking agent import), exports the full network, then passes it through `build_prompt` which internally calls `_build_beliefs_section`.

## Invariants

1. **Budget ceiling**: `agent_shown + local_shown <= max_beliefs` — total displayed beliefs never exceed the budget.
2. **Local floor**: `local_shown >= 5` — local beliefs always get at least 5 slots, regardless of agent pressure.
3. **Linear accumulation**: Agent belief count grows as N (number of beliefs shown), never N² — the core regression invariant.
4. **Local remainder formula**: `local_shown == max(5, max_beliefs - agent_total)` when there are enough local beliefs to fill the remainder.
5. **Deterministic sampling**: Same `seed` produces identical output across calls.

## Error Handling

Tests fail explicitly via `assert` with descriptive messages. The `_count_local_shown` and `_count_agent_shown` helpers assert the regex match is not `None` before extracting, providing the first 500 chars of output in the failure message for debugging. There's no exception handling — failures propagate as `AssertionError` to pytest.

## Topics to Explore

- [function] `reasons_lib/derive.py:_build_beliefs_section` — The function under test; read it to see the proportional budget calculation and the fixed counting logic
- [function] `reasons_lib/derive.py:build_prompt` — The public entry point that calls `_build_beliefs_section`; understanding its `stats` return value and how it determines agent vs. local partitioning
- [file] `reasons_lib/import_agent.py` — Creates the `:active` premise nodes and agent-prefixed beliefs that this test carefully avoids; understanding the import format explains why `_make_nodes` is shaped the way it is
- [diff] `issue-23-fix` — The actual diff that fixed the N² bug; seeing the one-line change (`count += len(belief_ids)` moved outside the inner loop) makes the regression tests concrete
- [general] `budget-proportional-allocation` — How `_build_beliefs_section` splits budget proportionally between agents and locals using `int(max_beliefs * agent_count / total_count)`

## Beliefs

- `derive-budget-local-floor-is-five` — `_build_beliefs_section` guarantees at least 5 local beliefs are shown regardless of agent budget pressure, via `max(5, max_beliefs - count)`
- `derive-budget-count-is-linear` — Agent belief counting in `_build_beliefs_section` accumulates as N (number of beliefs shown), not N² as in the pre-fix bug
- `derive-budget-tests-parse-output-headers` — Budget tests rely on regex-parsing `"showing N"` from section headers in the formatted output string, coupling them to the output format
- `make-nodes-excludes-active-premises` — The `_make_nodes` helper deliberately omits `:active` premise nodes because `_build_beliefs_section` matches agent beliefs via `startswith`, which would inflate counts

