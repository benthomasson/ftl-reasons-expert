---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_backtracking.py

# `tests/test_backtracking.py`

## Purpose

This is a pytest regression/specification suite for **dependency-directed backtracking** in the reasons graph.

It verifies that when a contradiction is represented as a `nogood`, the system:

1. Traces backwards through justification links to find the premises responsible.
2. Ranks candidate premises by disruption/entrenchment.
3. Retracts a premise rather than an arbitrary derived node.
4. Cascades retraction effects through dependent nodes.
5. Exposes the same behavior through the public `reasons.api` layer.

It is a test-only module: no production code imports it, and pytest discovers it automatically.

## Key Components

### `TestTraceAssumptions`

Tests `Network.trace_assumptions(node_id) -> list[str]`.

Contract enforced:

- A premise traces to itself: `trace_assumptions("a") == ["a"]`.
- A derived node traces recursively to its premise roots.
- Linear chains, diamonds, and multiple-root graphs are supported.
- Results are de-duplicated: in a diamond graph, `trace_assumptions("d") == ["a"]`, not `["a", "a"]`.
- For multiple roots, order is not assumed; tests use `set(...)`.
- Missing nodes raise `KeyError`.

### `TestFindCulprits`

Tests `Network.find_culprits(node_ids) -> list[dict]`.

Contract enforced:

- If a nogood contains premise nodes, those premises are culprits.
- If a nogood contains a derived node, the derived node is expanded to the premises supporting it.
- Each culprit dict has at least:
  - `"premise"`: the premise node ID.
  - `"would_resolve"`: nogood members that would be resolved if that premise were retracted.
- Culprits are sorted by dependent count: fewer dependents first, to minimize disruption.

### `TestBacktrackingInNogood`

Tests `Network.add_nogood(node_ids) -> changed`.

Contract enforced:

- `add_nogood` retracts a premise, not a derived node.
- Retracting a premise cascades dependents `OUT`.
- If both nogood nodes share the same premise, retracting that shared premise resolves both.
- A real backtrack writes a log entry with `action == "backtrack"` and a `"culprit"` field.
- If any nogood node is already `OUT`, `add_nogood` is a no-op: it returns `[]` and writes no backtrack log entry.

Important maintainer note: despite the name/docstring of `test_cascade_from_retracted_premise`, the actual assertions expect `"d"` to be retracted and `"c"`/`"b"` to stay `IN`. The docstring says `A → B → C` cascades `OUT`, but the test does not assert that.

### `TestEntrenchment`

Tests the private heuristic `Network._entrenchment(node_id) -> comparable score`.

Contract enforced:

- Higher score means more protected / less likely to be retracted.
- A sourced premise scores higher than an unsourced premise.
- A premise with `metadata={"beliefs_type": "OBSERVATION"}` scores higher than one with `"DERIVED"`.
- A node with more dependents scores higher than a less-depended node.
- In the “physics speculation” scenario, a sourced evidence premise survives while unsourced/derived speculation is retracted.

Some test names/docstrings are looser than the actual assertions. For example, `test_evidence_premise_more_entrenched_than_derived` actually asserts `evidence > basis`, where both are premises and only `evidence` has a source.

### `TestBacktrackingAPI`

Tests the SQLite-backed API wrapper in `reasons.api`.

Contract enforced:

- `api.init_db(db_path=...)`, `api.add_node(..., db_path=...)`, etc. use a per-test temporary database via `tmp_path`.
- API `add_node` accepts `sl="a"` as a shorthand support-link antecedent.
- `api.trace_assumptions("c", db_path=db)` returns:
  ```python
  {"node_id": "c", "premises": ["a"]}
  ```
- `api.find_culprits([...], db_path=db)` returns a dict with `"culprits"`.
- `api.add_nogood([...], db_path=db)` returns a dict containing `"backtracked_to"`; for a real backtrack it is not `None`.

## Patterns

- **Feature-grouped pytest classes**: each class owns one behavior area: trace, culprits, nogood handling, entrenchment, API.
- **Arrange-Act-Assert**: each test builds a small graph, invokes one operation, then asserts graph state or response shape.
- **White-box testing**: tests inspect `net.nodes[...].truth_value`, `net.log`, and call private `_entrenchment`.
- **Real objects, minimal mocking**: the suite uses real `Network` instances and a real temporary SQLite DB for API tests.
- **Set comparisons for unordered contracts**: multi-root traces and multi-culprit results use `set(...)` because ordering is not guaranteed.
- **Return-value coverage is partial**: many tests assign `changed = net.add_nogood(...)` but only `test_inactive_nogood_no_backtracking` asserts the exact return value `[]`. Most behavior is pinned through graph state.

## Dependencies

Imports:

- `pytest` for `pytest.raises` and `tmp_path`.
- `reasons.Justification` for constructing support links.
- `reasons.network.Network` for the core graph/truth-maintenance implementation under test.
- `reasons.api` for the persistent API layer.

Dependents:

- No production module imports this test file.
- pytest imports it during test discovery.
- The repository also contains `reasons_lib`, but this test imports from `reasons`, so the `reasons` package is the target.

## Flow

Core network tests typically execute:

1. Instantiate `Network()`.
2. Add premise nodes with `add_node(...)`.
3. Add derived nodes with `justifications=[Justification(type="SL", antecedents=[...])]`.
4. Call `trace_assumptions`, `find_culprits`, `add_nogood`, or `_entrenchment`.
5. Assert returned values, truth values, or log entries.

API tests execute:

1. Build a temp DB path with `tmp_path / "test.db"`.
2. Call `api.init_db(db_path=db)`.
3. Add nodes through `api.add_node(..., db_path=db)`.
4. Call the API function under test.
5. Assert the returned dict shape and preliminary values.

The expected `add_nogood` behavior is:

1. If any nogood node is already `OUT`, do nothing and return `[]`.
2. Otherwise, trace nogood nodes back to premises.
3. Rank candidate premises using dependent count and entrenchment.
4. Retract the least-protected / least-disruptive premise.
5. Cascade dependent nodes to `OUT`.
6. Log a `"backtrack"` entry containing the culprit.
7. Return some representation of changed nodes.

## Invariants

- New nodes default to `truth_value == "IN"`.
- `truth_value` values asserted here are the strings `"IN"` and `"OUT"`.
- A premise traces to itself.
- A derived node traces to one or more premises.
- Trace results are de-duplicated across multiple paths.
- Missing node IDs passed to `trace_assumptions` raise `KeyError`.
- `find_culprits` expands derived nogood members to their premises.
- `find_culprits` orders culprits by dependent count ascending.
- Backtracking retracts a premise, not a derived node directly.
- Retracting a shared premise cascades all dependents `OUT`.
- A nogood with any already-`OUT` node requires no backtracking.
- Entrenchment is monotonic with:
  - having a source,
  - `beliefs_type == "OBSERVATION"` over `"DERIVED"`,
  - having more dependents.

## Error Handling

- The only explicit error contract in this file is:
  ```python
  with pytest.raises(KeyError):
      net.trace_assumptions("missing")
  ```
- Inactive nogoods are handled gracefully as no-ops: `add_nogood` returns `[]` and writes no backtrack log.
- No error cases are covered for malformed API calls, missing DB paths, empty nogoods, cycles, duplicate node IDs, or invalid `Justification` inputs.
- Other exceptions, if raised by `Network` or `api`, are not caught in this file and would fail the test.

## Topics to Explore

- [file] `reasons/network.py` — Contains `Network`, including `trace_assumptions`, `find_culprits`, `add_nogood`, and `_entrenchment`.
- [function] `reasons/network.py:Network.add_nogood` — Inspect exact culprit ranking, retraction cascade, and log entry format.
- [function] `reasons/network.py:Network.find_culprits` — See how `would_resolve` is computed from the justification graph.
- [file] `reasons/api.py` — API wrappers, SQLite persistence, and response schemas used by `TestBacktrackingAPI`.
- [general] `entrenchment-heuristic` — Explore how source, `beliefs_type`, and dependent count combine into `_entrenchment`.

## Beliefs

- `trace-assumptions-returns-deduplicated-premise-ids` — `Network.trace_assumptions` returns premise IDs reached by recursively following justifications and returns the node itself when it is a premise.
- `find-culprits-orders-by-dependent-count` — `Network.find_culprits` returns culprits sorted so a premise with fewer dependents appears before one with more dependents.
- `add-nogood-retracts-premises-not-derived-nodes` — `Network.add_nogood` retracts a selected premise and cascades dependent derived nodes `OUT` rather than retracting a derived nogood member directly.
- `inactive-nogood-is-no-op` — If any node in a nogood is already `OUT`, `Network.add_nogood` returns an empty changed list and emits no backtrack log entry.
- `entrenchment-increases-with-source-observation-and-dependents` — `_entrenchment` scores sourced premises, `OBSERVATION` metadata, and nodes with more dependents above their unsourced/derived/less-depended counterparts.
