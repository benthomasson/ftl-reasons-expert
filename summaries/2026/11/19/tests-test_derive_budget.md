---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_derive_budget.py

# `tests/test_derive_budget.py`

## Purpose

This file is a focused regression-test module for the budget calculation in `reasons.derive._build_beliefs_section` and its integration through `build_prompt`.

Its central responsibility is guarding issue #23: the original bug incremented a budget counter with `count += len(belief_ids)` inside a per-belief loop, causing the count to grow quadratically (`N²`) instead of linearly (`N`). That inflated count starved the local-belief section because the remaining local budget is computed roughly as:

```python
remaining = max(5, max_beliefs - count)
```

The tests verify that agent belief counts accumulate linearly, that multi-agent counts sum correctly, that the local budget floor of 5 works, that edge cases do not crash, that sampling is deterministic, and that the full `build_prompt` path behaves correctly with a real temporary database.

## Key Components

### Module imports

- `re` — used to parse the rendered human-readable output sections.
- `pytest` — provides the `@pytest.fixture` decorator and test discovery.
- `reasons.api` — used in the database-backed integration test for `init_db`, `add_node`, and `export_network`.
- `reasons.derive._build_beliefs_section` — the private function under direct test.
- `reasons.derive.build_prompt` — the public integration path under test.

### `db(tmp_path)` fixture

```python
@pytest.fixture
def db(tmp_path):
    db_path = str(tmp_path / "reasons.db")
    api.init_db(db_path=db_path)
    return db_path
```

Creates an isolated SQLite database inside pytest’s temporary directory. The fixture returns the database path so tests can pass `db_path=db` into `api` functions. This keeps the integration test from touching any shared or persistent state.

### `_make_nodes(agent_beliefs, local_beliefs=None)`

Builds a synthetic node graph without a database.

Contract:

- `agent_beliefs`: `dict[str, list[str]]`, mapping agent name to belief suffixes.
- `local_beliefs`: optional list of local belief IDs.
- Returns `(nodes, derived, agents)`:
  - `nodes`: node ID -> metadata dict with `truth_value="IN"` and `text`.
  - `derived`: always `{}` for these synthetic tests.
  - `agents`: agent name -> list of generated node IDs.

The helper deliberately does not create `:active` premise nodes. Its docstring notes that `_build_beliefs_section` matches agent beliefs by `startswith`, so adding extra `agent-a:active` nodes would inflate agent belief counts in the function under test.

### `_count_local_shown(output)`

Extracts the number from output like:

```text
Local beliefs (20 beliefs, showing 15)
```

using:

```python
re.search(r"Local beliefs \(\d+ beliefs, showing (\d+)\)", output)
```

If the header is missing or reformatted, it raises an assertion with the first 500 characters of output to make failures debuggable.

### `_count_agent_shown(output, agent_name)`

Extracts the “showing N” count from an agent section like:

```text
Agent: agent-a (6 beliefs, showing 5)
```

It uses `re.escape(agent_name)` so agent names containing regex metacharacters are treated literally.

### Test classes

#### `TestCountLinearNotQuadratic`

The core regression tests.

- `test_six_beliefs_one_agent` expects `local_shown == 15` for 6 agent beliefs and 20 local beliefs with `max_beliefs=20`. The docstring explains the expected proportional budget:
  - `agent_budget = max(5, int(20 * 6 / 26)) = 5`
  - `count = 5`
  - `remaining = max(5, 20 - 5) = 15`
  - The old bug would produce `count = 5 * 5 = 25` and local budget `5`.
- `test_count_equals_n_not_n_squared` checks that `agent_shown + local_shown <= 20` for 10 agent beliefs and 50 local beliefs.

#### `TestMultiAgent`

Verifies accumulation across multiple agents.

- `test_two_agents_accumulate_correctly` asserts:
  ```python
  local_shown == max(5, 20 - agent_total)
  ```
  where `agent_total` is the sum of shown counts for both agents.
- `test_three_agents_budget_sums` does the same with three agents and `max_beliefs=30`.

These tests are important because the original `N²` bug would be worse with multiple agents.

#### `TestBudgetFloor`

Verifies the local section still gets at least 5 beliefs when agents consume most or all of the budget.

- `test_floor_of_five_when_agents_exceed_budget` uses 50 agent beliefs and 10 local beliefs with budget 20, asserting `local_shown >= 5`.
- `test_floor_applies_with_large_agent_set` uses 200 agent beliefs and asserts exactly `local_shown == 5`.

#### `TestEdgeCases`

Covers boundary conditions and smoke cases:

- `test_single_agent_belief`: `N=1` is a case where `N² == N`, so the old bug was invisible. It verifies the fix still allows `agent_shown == 1` and `local_shown == 10`.
- `test_no_agents`: builds local nodes only, calls `_build_beliefs_section(..., agents=None)`, and asserts no `"Agent:"` section appears.
- `test_no_local_beliefs`: agent-only network should omit `"Local beliefs"` and still include the agent section.
- `test_empty_network`: empty nodes and no agents should not crash and should return a string.
- `test_empty_network_with_agents`: agents dict may reference nodes that do not exist; the function should still return a string.
- `test_derived_beliefs_excluded_from_count`: constructs an agent node with a `justifications` field and a separate `derived` mapping. The test is somewhat weak: it only asserts that `"Agent: agent-a"` appears, but its name and setup document the intended invariant that derived beliefs should not inflate the budget count.

#### `TestSampling`

Tests `sample=True` mode.

- `test_sample_mode_respects_budget`: with 6 agent beliefs and 20 locals, sampling mode should still give locals at least 5.
- `test_sample_mode_deterministic`: calling `_build_beliefs_section` twice with the same `seed=99` must produce identical output.

#### `TestBuildPromptIntegration`

End-to-end coverage through the public prompt builder.

- Creates a real temporary database using `db`.
- Adds `agent-a:active`.
- Adds 6 imported agent beliefs under `sl="agent-a:active"`.
- Adds 20 local beliefs.
- Exports the network with `api.export_network(db_path=db)`.
- Calls `build_prompt(data["nodes"], budget=20)`.
- Asserts:
  - `stats["agents"] == 1`
  - `_count_local_shown(prompt) >= 5`
  - `_count_local_shown(prompt) <= 20`

This verifies that the budget fix survives the real database/export/prompt path.

## Patterns

### Regression-test classes grouped by concern

The file is organized into test classes that each isolate a concern:

- quadratic-vs-linear count
- multi-agent accumulation
- budget floor
- edge cases
- sampling
- end-to-end integration

This makes failures easy to localize.

### Synthetic graph construction instead of database setup

Most tests use `_make_nodes` to create in-memory `nodes`, `derived`, and `agents` dictionaries. This keeps unit tests fast and focused on `_build_beliefs_section`.

### Regex parsing as an output contract

The helpers `_count_local_shown` and `_count_agent_shown` treat the rendered prompt text as a contract. The expected headers are:

- `Local beliefs (N beliefs, showing M)`
- `Agent: <name> (N beliefs, showing M)`

Any change to those headers will fail these tests.

### Isolation via `tmp_path`

The integration test uses pytest’s `tmp_path` fixture to create a per-test SQLite database. No global state is shared.

### Deterministic sampling assertion

The sampling test explicitly pins reproducibility: same input and same seed must produce exactly equal output strings.

## Dependencies

### This file depends on

- `pytest` for fixtures and test execution.
- `reasons.api.init_db` — initializes a temporary database.
- `reasons.api.add_node` — inserts nodes into the temporary database.
- `reasons.api.export_network` — serializes network data back into a `{"nodes": ...}` structure.
- `reasons.derive._build_beliefs_section` — the private budget/rendering function under test.
- `reasons.derive.build_prompt` — the public prompt-building integration point.
- Exact output formatting from `_build_beliefs_section`, because the regex helpers parse the rendered headers.

### What depends on this file

This is a test module. It is not imported by production code. It is discovered and executed by pytest.

## Flow

1. Pytest collects the test classes and methods.
2. For tests that need a database, the `db` fixture creates a temporary SQLite database and calls `api.init_db`.
3. Most unit tests call `_make_nodes(...)` to construct synthetic `nodes`, `derived`, and `agents`.
4. Tests call `_build_beliefs_section(nodes, derived, agents, max_beliefs=...)`.
5. `_build_beliefs_section` returns a tuple; the tests unpack the rendered output and ignore the second value (`output, _ = ...`).
6. Regex helpers extract the “showing N” values from the output headers.
7. Assertions verify:
   - the agent count is linear,
   - local budget uses the correct remainder,
   - the floor of 5 applies,
   - edge cases render safely,
   - sampling is deterministic.
8. The integration test writes actual nodes to a temporary DB, exports the network, calls `build_prompt`, and validates both prompt content and `stats`.

## Invariants

- For `N` agent beliefs, the accumulated agent count must be `N`, not `N²`.
- The local budget is expected to follow:
  ```python
  remaining = max(5, max_beliefs - agent_count)
  ```
  The tests assert the floor of 5 in multiple scenarios.
- The sum of shown agent beliefs plus shown local beliefs should not exceed `max_beliefs` in the tested cases.
- If `agents=None`, the output must not contain an `"Agent:"` section.
- If there are no local beliefs, the output must not contain `"Local beliefs"`.
- Empty networks and missing agent node references must not crash.
- Sampling with the same seed must be deterministic.
- `build_prompt(...)` must return `(prompt, stats)` where `stats["agents"]` reflects the number of agents.
- `_make_nodes` intentionally avoids creating `:active` premise nodes because `_build_beliefs_section` matches agent beliefs by prefix/startswith behavior.

## Error Handling

- The regex helpers use `assert m is not None` with a short output snippet. If `_build_beliefs_section` changes its section headers, the test fails loudly with context.
- There is no explicit `try/except` in the tests. Any exception from `api.init_db`, `api.add_node`, `api.export_network`, `_build_beliefs_section`, or `build_prompt` propagates and fails the test.
- Database isolation is handled by pytest’s `tmp_path`, so failures should not leak state between tests.
- Several edge-case tests are smoke tests: they only assert that output is a string or that a section exists, rather than validating exact counts.

## Topics to Explore

- [file] `reasons/derive.py` — Contains `_build_beliefs_section` and `build_prompt`, including the actual budget allocation and rendering logic under test.
- [function] `reasons/derive.py:_build_beliefs_section` — Inspect how agent beliefs are matched, how counts accumulate, and how local/agent budgets are rendered.
- [function] `reasons/derive.py:build_prompt` — Understand how `nodes` are passed into `_build_beliefs_section` and how `stats["agents"]` is produced.
- [file] `reasons/api.py` — Review `init_db`, `add_node`, and `export_network` used by the integration test.
- [general] `issue-23-regression-math` — Trace the original `count += len(belief_ids)` bug and why it produced quadratic behavior and local-budget starvation.

## Beliefs

- `test-derive-budget-targets-issue-23` — The module docstring explicitly states these tests cover `_build_beliefs_section` budget calculation for issue #23.
- `_build_beliefs_section-count-is-linear-not-quadratic` — `TestCountLinearNotQuadratic` guards the regression where agent belief count must accumulate `N`, not `N²`.
- `local-budget-floor-is-five` — `TestBudgetFloor` asserts the local belief budget never drops below 5 when agents exceed the budget.
- `sample-mode-is-deterministic-with-same-seed` — `test_sample_mode_deterministic` asserts identical output for the same input and `seed=99`.
- `output-header-format-is-contract-for-regex-helpers` — `_count_local_shown` and `_count_agent_shown` parse exact header patterns such as `Local beliefs (N beliefs, showing M)` and `Agent: name (N beliefs, showing M)`.
