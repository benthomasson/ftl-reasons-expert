# File: tests/test_pg.py

**Date:** 2026-05-11
**Time:** 12:50

I'll work from the file content provided. Here's the explanation:

---

# `tests/test_pg.py` — PostgreSQL Backend Test Suite

## Purpose

This is the integration test suite for `PgApi`, the PostgreSQL-native storage backend for the ftl-reasons Truth Maintenance System. It validates that every TMS operation — node creation, retraction, assertion, propagation, search, export, and multi-tenancy — works correctly when backed by a real Postgres database rather than the default SQLite store (`reasons_lib/storage.py`).

It is a **behavioral contract test**: it doesn't test SQL queries or schema details, but rather the full API surface of `PgApi` through its public methods, verifying truth-value semantics, cascade behavior, and access control.

## Key Components

### Test Classes (22 total)

Each class maps 1:1 to a `PgApi` method or feature area:

| Class | API Under Test | What It Validates |
|---|---|---|
| `TestAddNode` | `add_node()` | Premise/derived creation, outlist blocking, duplicate rejection, access tag inheritance |
| `TestRefIntegrity` | `add_node()`, `add_justification()` | Referential integrity — phantom antecedent/outlist refs raise `KeyError` |
| `TestRetractNode` | `retract_node()` | Retraction, cascades, idempotency, reason metadata, missing-node errors |
| `TestAssertNode` | `assert_node()` | Restoration, cascade restoration, idempotency |
| `TestPropagation` | multiple | Diamond dependencies, outlist block/unblock/reblock, multiple justifications, retracted-pin semantics |
| `TestWhatIf` | `what_if_retract()`, `what_if_assert()` | Dry-run impact analysis — verifies results are correct **and** no mutation occurs |
| `TestChallenge` | `challenge()` | Dialectical challenge: auto-ID generation, collision handling, cascade, metadata |
| `TestDefend` | `defend()` | Dialectical defense: restoring challenged nodes, multi-challenge scenarios |
| `TestCompact` | `compact()` | Budget-constrained text summaries, access filtering, nogood inclusion, dependent-count sorting, summary-node elision |
| `TestExportNetwork` | `export_network()` | JSON export of full belief graph including justifications, nogoods, metadata, access filtering |
| `TestMultiTenancy` | `PgApi` with separate `project_id` | Project isolation — same node IDs in different projects don't collide |
| `TestSummarize` | `summarize()` | Summary nodes that depend on covered nodes |
| `TestSupersede` | `supersede()` | Belief replacement with reversibility |
| `TestNamespace*` (3 classes) | `ensure_namespace()`, namespace-aware `add_node()`/`add_justification()` | Multi-agent namespace prefixing, auto-created `:active` premise, cascade on agent deactivation |

### Fixture: `pg_api`

Defined in `tests/conftest.py`, injected into every test method. Creates an isolated `PgApi` instance with a unique project ID so tests don't interfere with each other or with persistent data.

### Module-level markers

```python
pytestmark = [pytest.mark.pg, skip_no_pg]
```

Every test in this file is marked `pg` (for selective test runs) and skipped if no PostgreSQL connection is available (controlled by `skip_no_pg` from conftest, likely checking for a `DATABASE_URL` env var).

## Patterns

**Fixture-per-test isolation.** Each test gets a fresh `pg_api` with an empty belief network. No test depends on state from a previous test — they're fully independent.

**Arrange-Act-Assert with API dictionaries.** Every test follows the same shape:
1. Set up nodes via `add_node()`
2. Perform an operation (retract, assert, challenge, etc.)
3. Assert on the returned dict and/or query with `show_node()`/`get_status()`

**No-mutation verification pattern.** The `TestWhatIf` class explicitly verifies that dry-run operations don't change database state — it runs the what-if, then checks `get_status()` to confirm counts are unchanged.

**Error-as-contract testing.** `KeyError` for missing nodes, `PermissionError` for access violations, `ValueError` for invalid operations (removing last justification, duplicate summary IDs) — these are tested as part of the API contract, not as edge cases.

**Direct SQL for setup only.** Two places bypass the API to write directly to Postgres (`TestCompact.test_compact_summary_nodes` and `TestPropagate.test_fixes_stale_truth`). These intentionally create inconsistent state that the API should detect and fix — testing self-healing behavior.

## Dependencies

**Imports:**
- `json` — parsing `search()` JSON output
- `pytest` — test framework, markers, `raises`
- `tests.conftest.skip_no_pg` — conditional skip marker
- `reasons_lib.pg.PgApi` — only in `TestMultiTenancy`, which constructs a second API instance directly
- `os`, `uuid` — only in `TestMultiTenancy` for `DATABASE_URL` and unique project IDs

**Depended on by:** Nothing imports this file. It's a leaf in the dependency graph — consumed only by pytest.

## Flow

Tests exercise the full TMS lifecycle:

1. **Node creation** → premises (no justifications) and derived nodes (with `sl` antecedents and optional `unless` outlist)
2. **Truth propagation** → retract a premise, watch dependents cascade to OUT; assert it back, watch them return to IN
3. **Non-monotonic reasoning** → outlist nodes block derived truth; retracting the blocker unblocks; re-asserting re-blocks
4. **Dialectical reasoning** → `challenge()` creates a challenge node whose outlist includes the target; `defend()` creates a defense node whose outlist includes the challenge
5. **Query operations** → `search()`, `get_status()`, `list_nodes()`, `list_gated()`, `explain_node()`, `trace_assumptions()`, `compact()`
6. **Structural mutations** → `add_justification()`, `remove_justification()`, `convert_to_premise()`, `supersede()`, `summarize()`
7. **Multi-agent** → namespace-prefixed node IDs, auto-created `:active` premises, agent deactivation cascades

## Invariants

- **Premises are always IN when created.** Every `add_node()` without `sl` yields `truth_value == "IN"`.
- **Derived nodes reflect antecedent truth.** If all `sl` antecedents are IN and all `unless` outlist nodes are OUT, the derived node is IN; otherwise OUT.
- **Retraction cascades transitively.** Retracting a root premise makes all transitive dependents go OUT.
- **Assertion restores transitively** — but only non-pinned nodes. A node explicitly retracted (pinned OUT) stays OUT even when its antecedent returns to IN (`test_retracted_pin_skipped`).
- **Referential integrity on creation.** Phantom antecedents or outlist references raise `KeyError` — you can't create edges to nonexistent nodes.
- **Last justification cannot be removed.** `remove_justification()` on a node with exactly one justification raises `ValueError`.
- **What-if operations never mutate.** `what_if_retract()`/`what_if_assert()` return cascade predictions without side effects.
- **Access tags propagate upward.** A derived node inherits the union of its antecedents' access tags.
- **Namespace auto-prefixing.** Passing `namespace="agent1"` to `add_node("x", ...)` produces node ID `"agent1:x"`, but `"agent1:x"` is not double-prefixed.
- **Projects are fully isolated.** Two `PgApi` instances with different project IDs can use the same node IDs without collision.

## Error Handling

The test suite validates these error semantics of `PgApi`:

| Condition | Exception | Verified In |
|---|---|---|
| Node not found | `KeyError` | `test_retract_not_found`, `test_show_not_found`, `test_add_nogood_not_found`, etc. |
| Phantom antecedent/outlist reference | `KeyError` (with match on missing ID) | `TestRefIntegrity` — `match="ghost"`, `match="x.*y\|y.*x"` |
| Access denied (wrong `visible_to` tags) | `PermissionError` | `test_show_access_denied`, `test_visible_to_denied` |
| Duplicate node ID | generic `Exception` | `test_add_duplicate_raises` |
| Remove last justification | `ValueError` with `match="only one justification"` | `test_remove_last_raises` |
| Remove justification from premise | `ValueError` with `match="premise"` | `test_remove_premise_raises` |
| Defense ID collision | `ValueError` with `match="Defense node"` | `test_defend_duplicate_id_raises` |
| Invalid justification index | `IndexError` | `test_remove_invalid_index` |

---

## Topics to Explore

- [file] `reasons_lib/pg.py` — The implementation under test: PostgreSQL-backed TMS with all the SQL, propagation logic, and access control
- [file] `tests/conftest.py` — Defines `pg_api` fixture, `skip_no_pg` marker, and database cleanup strategy
- [file] `tests/test_pg_dispatch.py` — Likely tests the dispatch/routing layer that sits above PgApi
- [function] `reasons_lib/pg.py:PgApi.propagate` — The truth-propagation engine that `TestPropagate` exercises by manually corrupting DB state
- [general] `sqlite-vs-pg-parity` — Whether `test_pg.py` and `test_storage.py`/`test_api.py` enforce the same behavioral contract, or if PgApi has features the SQLite backend lacks

## Beliefs

- `pg-api-tests-require-database-url` — All tests in `test_pg.py` are skipped unless a PostgreSQL connection is available (via `skip_no_pg` marker)
- `pg-retracted-pin-prevents-cascade-restore` — A node explicitly retracted stays OUT when its antecedent is re-asserted, even though the justification chain is satisfied (tested in `test_retracted_pin_skipped`)
- `pg-what-if-is-read-only` — `what_if_retract()` and `what_if_assert()` return predicted cascades without modifying any database state (verified by post-operation `get_status()` checks)
- `pg-namespace-auto-creates-active-premise` — Calling `add_node()` with a `namespace` parameter auto-creates an `<ns>:active` premise node and adds it to the derived node's antecedent list
- `pg-challenge-uses-outlist-mechanism` — Dialectical `challenge()`/`defend()` work via outlist-based non-monotonic reasoning, not by directly toggling truth values — defending all challenges restores the target to IN

