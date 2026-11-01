---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_pg.py

# `tests/test_pg.py` — PostgreSQL Storage Backend Integration Tests

## Purpose

This is the **integration test suite** for the `PgApi` class in `reasons/pg.py` — the PostgreSQL-native storage backend for a **defeasible (non-monotonic) belief network**. The project ("reasons") implements a formal reasoning engine where beliefs (nodes) are either `IN` (active) or `OUT` (retracted), and truth values propagate through justification chains (SL rules) and defeaters (outlists).

This file owns the responsibility of verifying that the PG backend correctly implements:

- Node lifecycle (add, retract, assert, challenge, defend, supersede)
- Truth-value propagation (cascading IN/OUT through dependency graphs)
- Referential integrity (no dangling antecedents/outlists)
- Multi-tenancy (project-level data isolation)
- Namespacing (agent-scoped belief networks)
- Access control (tag-based visibility)
- Nogood learning and backtracking
- Query/search/export surfaces

It is an **integration test** (not unit) — it requires a live PostgreSQL instance and exercises the full SQL query path, not just Python logic.

## Key Components

The file is organized into ~25 test classes, each targeting a distinct API surface of `PgApi`:

| Class | Responsibility |
|-------|---------------|
| `TestAddNode` | Creating premises and derived nodes; access tag inheritance |
| `TestRefIntegrity` | Rejecting references to non-existent nodes (KeyError) |
| `TestRetractNode` / `TestAssertNode` | Toggling truth values with cascade |
| `TestPropagation` | Diamond dependencies, outlist gating, multi-justification |
| `TestWhatIf` | **Non-mutating** hypothetical retract/assert (read-only simulation) |
| `TestChallenge` / `TestDefend` | Dialectical challenge/defense cycle |
| `TestNogood` / `TestFindCulprits` | Backtracking: learn nogoods, find least-entrenched premises |
| `TestCompact` | Token-budgeted summary rendering for LLM context |
| `TestExportNetwork` | Full JSON serialization of the belief graph |
| `TestMultiTenancy` | Verifying `domain_id` partitioning between projects |
| `TestEnsureNamespace` / `TestListNamespaces` / `TestNamespaceAddNode` | Agent-scoped sub-networks with auto-creation of premise anchors |
| `TestTraceAccessTags` | Walking the justification graph to collect cumulative access tags |

The single external dependency within the file body is `PgApi` itself (imported directly in `TestMultiTenancy` to spin up a second project). Everything else goes through the `pg_api` fixture.

## Patterns

- **Conditional execution via `pytestmark`**: The module-level `pytestmark = [pytest.mark.pg, skip_no_pg]` means these tests are tagged for selective runs (`-m pg`) and auto-skip if PostgreSQL is unavailable. This keeps CI green when only SQLite or in-memory backends are available.

- **Arrange → Act → Assert**: Every test follows this structure. The "arrange" phase builds a small belief graph (2–5 nodes), the "act" calls one API method, and the "assert" checks the returned dict and/or re-queries via `show_node`.

- **Direct SQL in tests**: A few tests (`TestCompact::test_compact_summary_nodes`, `TestPropagate::test_fixes_stale_truth`) execute raw SQL through `pg_api.conn.cursor()` to set up states the API doesn't expose (e.g., corrupting a truth value to test `propagate()` repair). This signals the fixture exposes the raw `psycopg2` connection.

- **Non-mutation verification**: `TestWhatIf` tests always call `what_if_*` and then immediately assert the real state is unchanged. This is a critical contract: what-if is a **pure simulation**.

- **Collision/edge-case naming**: Tests like `test_challenge_auto_id_collision` and `test_defend_duplicate_id_raises` pin down the exact auto-generation and error semantics for node IDs.

## Dependencies

**Imports (this file → other modules):**
- `pytest` — test runner, `pytest.raises`, markers
- `json` — parsing JSON-formatted search results
- `tests.conftest.skip_no_pg` — conditional skip helper
- `reasons.pg.PgApi` — the class under test (only in `TestMultiTenancy`)
- `os`, `uuid` — for multi-tenancy setup

**What depends on this file:**
- The CI pipeline (`.github/workflows/`) — likely invokes `pytest -m pg` or the full suite with a PG service container
- `tests/test_pg_dispatch.py` — likely tests the dispatch layer that routes between backends (SQLite vs PG); this file validates the PG path
- No production code imports this file (it's a leaf in the dependency graph)

**The `pg_api` fixture** (defined in `tests/conftest.py`, not shown) is the critical dependency. It:
1. Connects to a PostgreSQL instance (via `DATABASE_URL` env var)
2. Creates an isolated `domain_id` per test (or test class)
3. Yields a `PgApi` instance
4. Tears down (drops or truncates tables) after each test

## Flow

Each test class is independent; within a class, tests are also independent (no shared mutable state between test methods). A typical execution path:

```
pytest collects TestAddNode::test_add_premise
  → conftest.py creates fresh PgApi with new domain_id
  → test calls pg_api.add_node("a", "Alpha premise")
    → PgApi executes INSERT into rms_nodes
    → returns {"node_id": "a", "truth_value": "IN", "type": "premise"}
  → assertions check the dict
  → conftest teardown: DELETE FROM rms_nodes WHERE domain_id = ...
```

For propagation tests, the flow is:

```
add a (IN) → add b (SL=a, IN) → retract a
  → PgApi.retract_node("a"):
      1. SET a.truth_value = 'OUT'
      2. BFS/DFS over dependents: b → check if any justification still holds
      3. b's only justification is SL=a, a is OUT → b goes OUT
      4. Recurse to b's dependents
  → returns {"changed": ["a","b"], "went_out": ["a","b"]}
```

The `TestWhatIf` path is different: it runs the same cascade logic in a **read-only transaction** (or a shadow computation) and returns the hypothetical result without committing.

## Invariants

These are rules the tests **pin down** and that `PgApi` must always uphold:

1. **Referential integrity**: `add_node` with an `sl` or `unless` referencing a non-existent node raises `KeyError` with the ghost ID in the message. Premises (no `sl`) bypass this check.

2. **Truth-value determinism**: A node's truth value is a pure function of (a) its justifications' antecedent states, (b) its outlist's states, and (c) whether it's a premise. No other factor matters.

3. **Cascade completeness**: Retraction/assertion must propagate to *all* transitive dependents. The diamond test (`a → b, c → d`) ensures both paths are covered.

4. **Idempotency**: Retracting an already-OUT node or asserting an already-IN node returns `changed: []` — no spurious side effects.

5. **Multi-tenancy isolation**: Two `PgApi` instances with different `domain_id` values see completely disjoint node sets, even for identical node IDs.

6. **Access tag monotonicity**: A derived node's effective access tags are the **union** of its antecedents' tags (test: `test_access_tag_inheritance`). This means visibility only tightens as you go up the chain.

7. **What-if purity**: `what_if_retract` / `what_if_assert` never mutate state. Tests verify this by calling `get_status()` afterward.

8. **Namespace auto-creation**: Calling `add_node(..., namespace="X")` implicitly ensures `X:active` exists as an IN premise. The derived node depends on both its explicit `sl` and `X:active`.

## Error Handling

The tests define the **error contract** of `PgApi`:

| Condition | Exception | Message hint |
|-----------|-----------|-------------|
| Node not found (show, retract, assert, explain) | `KeyError` | — |
| Phantom antecedent/outlist reference | `KeyError` | Contains the ghost ID (e.g., `"ghost"`) |
| Multiple missing references | `KeyError` | Contains all missing IDs |
| Duplicate node ID | `Exception` (unspecified type) | — |
| Access tag denial | `PermissionError` | — |
| Remove last justification | `ValueError` | `"only one justification"` |
| Remove justification from premise | `ValueError` | `"premise"` |
| Invalid justification index | `IndexError` | — |
| Duplicate defense/challenge ID | `ValueError` | `"Defense node"` |
| Duplicate summary ID | `ValueError` | — |

Notably, `test_add_duplicate_raises` catches a bare `Exception` rather than a specific type — this suggests the duplicate check may not yet be a well-typed error, or the test is being permissive. A maintainer tightening this contract should pin it to `ValueError` or a custom `DuplicateNodeError`.

The `TestMultiTenancy` test does manual cleanup in a `finally` block, iterating over 5 tables by name — this is fragile if schema changes add tables.

## Topics to Explore

- [file] `tests/conftest.py` — Defines the `pg_api` fixture, `skip_no_pg` helper, and the PG connection lifecycle; understanding the isolation strategy (per-test vs per-class `domain_id`) is essential before modifying any test
- [file] `reasons/pg.py` — The class under test; specifically the `propagate` method's BFS algorithm and how it handles outlist re-evaluation during cascade
- [file] `tests/test_pg_dispatch.py` — Tests the routing layer that selects between SQLite and PG backends; reveals how `PgApi` plugs into the broader `storage.py` abstraction
- [function] `reasons/pg.py:what_if_retract` — The non-mutating simulation is the most complex method tested here; understanding how it shadows the cascade without committing is key to reasoning about correctness
- [general] `defeasible-reasoning-semantics` — The SL/UL justification types, nogood learning, and entrenchment ordering follow a specific formal logic (similar to ASP or TMS); the `CHANGELOG.md` or `README.md` likely documents the intended semantics that these tests encode

## Beliefs

- `pg-backend-enforces-referential-integrity` — `PgApi.add_node` and `PgApi.add_justification` raise `KeyError` (not silently create or ignore) when `sl` or `unless` references a node ID that does not exist in the current `domain_id`
- `what-if-operations-produce-no-mutation` — `what_if_retract` and `what_if_assert` are pure simulations; calling them leaves `get_status()`, node truth values, and the propagation log completely unchanged
- `access-tags-accumulate-upward` — A derived node's effective access tag set is the union of all tags in its transitive antecedent chain; visibility only tightens (narrows) as you move from premises to conclusions
- `namespace-add-extends-justification-with-active-premise` — When a node is added via `namespace="X"`, its justification implicitly includes `X:active` as an additional antecedent, so retracting the namespace anchor cascades to all nodes in that namespace
- `pg-multi-tenancy-partitions-by-domain-id` — All tables (`rms_nodes`, `rms_justifications`, `rms_nogoods`, etc.) are partitioned by a `domain_id` column; two `PgApi` instances with different `domain_id` values are fully isolated, even for identical node IDs
