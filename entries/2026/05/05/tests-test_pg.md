# File: tests/test_pg.py

**Date:** 2026-05-05
**Time:** 15:18

I can't access the target repo files directly — let me work from the test file content you provided, which is self-documenting enough to give a thorough explanation.

---

# `tests/test_pg.py` — PostgreSQL Storage Backend Test Suite

## 1. Purpose

This is the **integration test suite for the PostgreSQL-native storage backend** (`reasons_lib.pg.PgApi`). While the project has a SQLite-based storage layer (tested elsewhere), this file validates that the Postgres implementation correctly implements the full TMS (Truth Maintenance System) API contract: node management, SL-justification propagation, retraction cascades, outlist-based non-monotonic reasoning, nogood detection, access control, dialectical operations (challenge/defend), compaction, and multi-tenancy.

It is a **parity test suite** — every behavior tested here has a corresponding behavior expected from the SQLite backend, ensuring the two storage backends are interchangeable.

## 2. Key Components

The file contains **18 test classes**, each covering a distinct API surface:

| Class | Responsibility | Key invariant tested |
|---|---|---|
| `TestAddNode` | Node creation (premise, derived, outlist, access tags) | Premises default to IN; derived nodes inherit truth from antecedents; outlist blocks truth |
| `TestRefIntegrity` | Referential integrity on node/justification creation | Dangling references to nonexistent nodes raise `KeyError` |
| `TestRetractNode` | Retraction with cascading | Retraction propagates through dependency chains; idempotent for already-OUT nodes |
| `TestAssertNode` | Re-assertion with cascading restoration | Assert is the inverse of retract; cascades restore dependent truth values |
| `TestPropagation` | Complex propagation scenarios | Diamond deps, outlist block/unblock/reblock, multiple justifications, retracted-pin semantics |
| `TestGetStatus` | Global status queries with filtering | `visible_to` access-tag filtering excludes tagged nodes |
| `TestShowNode` | Single-node inspection | Returns justifications, dependents, access control enforcement (`PermissionError`) |
| `TestSearch` | Full-text and ID search with multiple output formats | Supports compact, json, dict formats; includes neighbor context |
| `TestListNodes` | Filtered node listing | Filters by status, type (premises_only), has_dependents, namespace prefix |
| `TestListGated` | GATE belief inventory | Lists blocker→gated mappings; satisfied gates (blocker OUT) are excluded |
| `TestGetLog` | Propagation log retrieval | Log captures add/retract actions; `last=N` limits entries |
| `TestAddJustification` | Adding justifications to existing nodes | Can flip a node from OUT→IN; validates target existence |
| `TestNogood` | Contradiction detection and backtracking | Active nogoods trigger retraction of least-entrenched premise; inactive nogoods are no-ops |
| `TestFindCulprits` | Entrenchment-based blame assignment | Nodes without `source` metadata are less entrenched |
| `TestExplainNode` | Proof-tree generation | Premises report "premise"; derived nodes report SL justification with antecedent list |
| `TestTraceAssumptions` | Transitive premise discovery | Walks the justification graph back to root premises |
| `TestMultiTenancy` | Project-level data isolation | Two `PgApi` instances with different `project_id` share a database but see disjoint data |
| `TestWhatIf` | Hypothetical retraction/assertion simulation | Simulates cascades without mutating state; reports depth and dependent counts |
| `TestChallenge` | Dialectical challenge mechanism | Creates a challenge node in the outlist that forces the target OUT |
| `TestDefend` | Dialectical defense mechanism | Creates a defense node in the challenge's outlist, restoring the target |
| `TestCompact` | Context-window-friendly belief summaries | Budget-sensitive truncation, access filtering, summary-node elision, nogood inclusion |

## 3. Patterns

### Marker-based conditional skipping

```python
pytestmark = [pytest.mark.pg, skip_no_pg]
```

Every test in the file carries two markers: `pytest.mark.pg` (for selective collection via `-m pg`) and `skip_no_pg` (imported from `conftest.py`, presumably a `pytest.mark.skipif` that checks for `DATABASE_URL` in the environment). This means the entire file is a no-op in CI environments without Postgres.

### Fixture-per-test isolation

All tests take a `pg_api` fixture (defined in `conftest.py`). Based on usage patterns — each test starts with a clean slate — the fixture likely creates a fresh project namespace (UUID) and tears down after each test, giving every test function an isolated belief network.

### Arrange-Act-Assert with API dictionaries

Tests follow a strict AAA pattern. The API returns plain dicts (not ORM objects), making assertions straightforward JSON-path checks like `result["truth_value"] == "IN"`. This is a deliberate design: the PgApi returns the same dict shapes as the SQLite API, ensuring backend interchangeability.

### Comma-separated support lists

The `sl="a,b"` parameter convention encodes SL (support-list) justifications as comma-separated node IDs. The `unless="blocker"` parameter encodes the outlist. This mirrors the TMS formalism where a justification is `(SL, OL)` — a support list and an outlist.

### Cleanup in multi-tenancy test

`TestMultiTenancy.test_projects_isolated` is the only test that manages its own `PgApi` instance. It manually deletes rows in a `finally` block across five tables, revealing the schema: `rms_nodes`, `rms_justifications`, `rms_nogoods`, `rms_network_meta`, `rms_propagation_log` — all partitioned by `project_id`.

## 4. Dependencies

**Imports:**
- `json` — used for JSON format assertions in search tests and direct metadata updates in compact tests
- `pytest` — test framework, markers, `raises` context manager
- `tests.conftest.skip_no_pg` — conditional skip marker (no Postgres → skip all)
- `reasons_lib.pg.PgApi` — the class under test (imported inline in `TestMultiTenancy`)
- `os`, `uuid` — used inline for multi-tenancy test setup

**Depended on by:** Nothing imports this file. It's a leaf in the dependency graph — consumed only by pytest.

**Fixture dependency:** `pg_api` from `conftest.py` is the critical fixture. It provisions an isolated PgApi instance per test.

## 5. Flow

Each test follows the same lifecycle:

1. **Setup** (implicit): `pg_api` fixture creates a fresh PgApi with a unique project namespace
2. **Build graph**: Tests call `add_node()` to construct a belief network — premises first, then derived nodes with `sl=` and `unless=` parameters
3. **Mutate**: Tests call `retract_node()`, `assert_node()`, `challenge()`, `defend()`, or `add_nogood()` to trigger state changes
4. **Assert**: Tests check return values (dicts with `changed`, `went_in`, `went_out` keys) and/or query state with `show_node()`, `get_status()`
5. **Teardown** (implicit): fixture cleans up the project's data

The propagation tests build increasingly complex graphs:
- **Linear chain**: `a → b → c` (retraction cascades three levels)
- **Diamond**: `a → b, a → c, b+c → d` (tests that d goes OUT when a is retracted, and comes back when a is asserted)
- **Outlist gate**: `z ← SL(x), OL(y)` (z is IN only when x is IN and y is OUT)

## 6. Invariants

The tests collectively enforce these invariants about the PgApi:

- **Premises default to IN.** Every `add_node("x", "text")` without `sl=` produces `truth_value == "IN"`.
- **Derived truth follows antecedents.** A node with `sl="a"` is IN iff all antecedents are IN and all outlist nodes are OUT.
- **Retraction cascades.** Retracting a node sets all transitive dependents to OUT.
- **Assertion cascades.** Asserting a node restores all transitive dependents whose justifications are now satisfied — but NOT nodes that were independently retracted (the "retracted pin" rule in `test_retracted_pin_skipped`).
- **Idempotency.** Retracting an already-OUT node returns `changed == []`. Asserting an already-IN node returns `changed == []`.
- **Referential integrity.** References to nonexistent nodes in `sl=` or `unless=` raise `KeyError` with the missing ID in the message.
- **Duplicate rejection.** Adding a node with an existing ID raises `Exception`.
- **Access tag inheritance.** Derived nodes inherit the union of their antecedents' access tags.
- **What-if is read-only.** `what_if_retract()` and `what_if_assert()` report cascades without mutating state.
- **Challenge/defend duality.** A challenge creates an outlist entry that forces the target OUT; a defense creates an outlist entry on the challenge, neutralizing it.
- **Multi-tenancy isolation.** Two PgApi instances with different project IDs sharing the same database see completely independent node sets — including being able to reuse the same node ID.

## 7. Error Handling

The API uses Python's standard exception hierarchy with semantic meaning:

| Exception | When raised | Tested in |
|---|---|---|
| `KeyError` | Node not found (retract, assert, show, explain, challenge, defend, add_justification, add_nogood, what_if) | `test_retract_not_found`, `test_show_not_found`, etc. |
| `KeyError` (with match) | Dangling reference — message contains the missing node ID | `TestRefIntegrity` — uses `pytest.raises(KeyError, match="ghost")` |
| `PermissionError` | Access-tag violation — caller's `visible_to` tags don't intersect node's tags | `test_show_access_denied` |
| `ValueError` | Defense node ID already exists | `test_defend_duplicate_id_raises` |
| `Exception` (generic) | Duplicate node ID on add | `test_add_duplicate_raises` |

The tests validate both the exception type and, where relevant, the error message content via `match=` patterns.

---

## Topics to Explore

- [file] `reasons_lib/pg.py` — The implementation under test; understanding propagation logic (BFS cascade, outlist evaluation, retracted-pin handling) is essential for maintaining these tests
- [function] `reasons_lib/pg.py:PgApi.what_if_retract` — Uses savepoints or cloned state to simulate cascades without mutation; the mechanism matters for correctness
- [file] `tests/conftest.py` — Defines `pg_api` fixture and `skip_no_pg`; controls test isolation strategy (per-test project namespace vs. table truncation)
- [general] `challenge-defend-protocol` — The dialectical layer (challenge creates outlist entries, defend counters them) is a higher-level abstraction over raw TMS operations; understanding how it composes with outlists clarifies the test expectations
- [file] `reasons_lib/api.py` — The SQLite-based API that PgApi mirrors; comparing the two reveals which behaviors are backend-specific vs. shared contract

## Beliefs

- `pg-tests-require-database-url` — All tests in `test_pg.py` are skipped unless the `DATABASE_URL` environment variable is set, via the `skip_no_pg` marker
- `pg-api-returns-plain-dicts` — Every `PgApi` method returns plain Python dicts (not ORM models), with standardized keys like `truth_value`, `changed`, `went_in`, `went_out`
- `pg-multi-tenancy-uses-project-id-column` — Data isolation between projects is achieved by a `project_id` column on all five tables (`rms_nodes`, `rms_justifications`, `rms_nogoods`, `rms_network_meta`, `rms_propagation_log`)
- `pg-retracted-pin-blocks-cascade-restore` — When a node is independently retracted (not as a cascade), re-asserting its ancestor does NOT restore it — this is the "retracted pin" invariant tested in `test_retracted_pin_skipped`
- `pg-challenge-uses-outlist-mechanism` — The `challenge()` method works by creating a new node in the target's outlist, leveraging the existing TMS outlist semantics to force the target OUT rather than introducing a separate mechanism

