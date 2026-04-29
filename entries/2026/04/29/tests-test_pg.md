# File: tests/test_pg.py

**Date:** 2026-04-29
**Time:** 17:16

I'll work from the file content you provided. The test file is thorough enough to reverse-engineer the API contract.

---

# `tests/test_pg.py` — PostgreSQL Storage Backend Test Suite

## 1. Purpose

This file is the integration test suite for `PgApi`, a PostgreSQL-native implementation of the reason maintenance system (TMS) storage backend. It validates that the full TMS lifecycle — node creation, truth propagation, retraction cascades, outlist-based non-monotonic reasoning, nogood detection, dialectical challenge/defend, compaction, and multi-tenancy — works correctly when backed by a real PostgreSQL database.

It mirrors the behavioral contract that the SQLite-based backend (`storage.py` / `api.py`) must also satisfy, but exercises the Postgres-specific implementation in `reasons_lib/pg.py`.

## 2. Key Components

The file contains **16 test classes**, each covering a distinct API surface area:

| Class | What it tests |
|---|---|
| `TestAddNode` | Node creation — premises, derived nodes, outlist, duplicates, access tags, tag inheritance |
| `TestRetractNode` | Retraction — single nodes, cascades, idempotency, reason metadata, missing nodes |
| `TestAssertNode` | Re-assertion — restoring retracted nodes, cascade restoration, idempotency |
| `TestPropagation` | Truth propagation — diamond dependencies, outlist blocking/unblocking/reblocking, multiple justifications, retracted-pin semantics |
| `TestGetStatus` | Status queries — empty state, node counts, `visible_to` access filtering |
| `TestShowNode` | Node inspection — premises, derived with justifications, dependents, not-found, access denied |
| `TestSearch` | Full-text search — by text, by ID, no results, compact/json/dict formats, neighbor inclusion |
| `TestListNodes` | Listing — all, by status, premises-only, has-dependents, by namespace |
| `TestListGated` | GATE belief listing — no gates, active gates, satisfied gates, multiple gated per blocker, blocker text |
| `TestGetLog` | Propagation log — entries after add, `last=N` filtering |
| `TestAddJustification` | Adding secondary justifications — truth value changes, missing nodes |
| `TestNogood` | Contradiction recording — active (triggers backtracking), inactive, missing nodes |
| `TestFindCulprits` | Entrenchment analysis — selecting least-entrenched premise for retraction |
| `TestExplainNode` | Explanation traces — premises, derived IN, derived OUT with failed antecedents |
| `TestTraceAssumptions` | Assumption tracing — premise self-trace, chains, diamond convergence |
| `TestMultiTenancy` | Project isolation — same node IDs in different projects don't collide |
| `TestWhatIf` | Hypothetical reasoning — dry-run retract/assert with cascade analysis, no mutation guarantee, outlist restoration |
| `TestChallenge` | Dialectical challenge — creates challenge node, retracts target, cascades, custom IDs, collision handling |
| `TestDefend` | Dialectical defense — creates defense node that defeats a challenge, restores target, handles multiple challenges |
| `TestCompact` | Context compaction — generates a budget-constrained markdown summary with access filtering, nogood inclusion, dependent-count sorting, and summary-node elision |

## 3. Patterns

**Module-level markers.** The `pytestmark` list applies two markers to every test:
- `pytest.mark.pg` — allows selective test execution (`pytest -m pg`)
- `skip_no_pg` — imported from `conftest.py`, presumably a `pytest.mark.skipif` that skips the entire module when no PostgreSQL connection (`DATABASE_URL`) is available

**Fixture-based isolation.** Every test method takes `pg_api` as its sole fixture. Based on the `TestMultiTenancy` class (which constructs a second `PgApi` with `os.environ["DATABASE_URL"]` and a fresh `uuid` project ID), we can infer:
- `pg_api` is a `PgApi` instance scoped per test
- Each test gets a unique `project_id` so tests don't contaminate each other
- The fixture handles `init_db()` setup and row cleanup on teardown

**Arrange-Act-Assert.** Tests follow a strict pattern: set up nodes with `add_node`, perform an operation, assert on the returned dict and/or query the state with `show_node` / `get_status`.

**No-mutation verification.** The `TestWhatIf` tests explicitly verify that `what_if_retract` and `what_if_assert` are read-only operations — they check that the database state is unchanged after the call. This is a critical invariant for hypothetical reasoning.

**Dialectical protocol.** Challenge/defend follows a structured pattern: `challenge()` creates a node that acts as an outlist blocker on the target; `defend()` creates a node that blocks the challenge. Multiple challenges require multiple defenses before the target is restored.

## 4. Dependencies

**Imports:**
- `json` — used for parsing `search(format="json")` results and setting metadata directly via SQL
- `pytest` — test framework, markers, `raises`
- `tests.conftest.skip_no_pg` — conditional skip marker
- `reasons_lib.pg.PgApi` — the system under test (imported inside `TestMultiTenancy` only; elsewhere via the `pg_api` fixture)
- `os`, `uuid` — used only in `TestMultiTenancy` for constructing a second isolated API instance

**Depended on by:** Nothing imports this file. It's a leaf in the dependency graph — pure test code.

## 5. Flow

A typical test's execution flow:

1. pytest injects `pg_api` — a `PgApi` connected to Postgres with a unique `project_id`
2. Test builds a belief network by calling `add_node()` with SL justifications (`sl="a,b"`) and outlists (`unless="blocker"`)
3. Test triggers a state change (`retract_node`, `assert_node`, `challenge`, `defend`, `add_nogood`)
4. The PgApi implementation propagates truth values through the dependency graph — this is the core TMS algorithm happening inside Postgres
5. Test inspects the result dict (which contains `changed`, `went_in`, `went_out` fields) and/or queries individual nodes with `show_node()`
6. Fixture teardown deletes all rows for the test's `project_id`

**Data transformations of note:**
- `add_node(id, text, sl="a,b", unless="blocker")` — the `sl` parameter is a comma-separated string of antecedent IDs; `unless` names an outlist node
- `search(query, format="dict")` returns a structured dict with `results`, `count`, and `neighbors`
- `compact(budget=N, visible_to=[...])` returns a markdown string summarizing the belief network, truncated to fit a token budget

## 6. Invariants

These are the behavioral invariants this test suite enforces:

1. **Premises are IN on creation** — `add_node` without `sl` creates a premise with `truth_value == "IN"`
2. **Derived truth tracks antecedents** — a derived node is IN only if all its SL antecedents are IN
3. **Outlist semantics** — a node with `unless="X"` is OUT when X is IN, IN when X is OUT (and all SL antecedents are IN)
4. **Retraction cascades** — retracting a premise propagates OUT through all transitive dependents
5. **Assertion cascades** — re-asserting a premise propagates IN back through dependents (respecting outlists)
6. **Retracted pins are sticky** — if a node is explicitly retracted (`test_retracted_pin_skipped`), re-asserting its antecedent does NOT restore it
7. **Multiple justifications provide resilience** — a node with two SL justifications stays IN if either is satisfied
8. **Duplicate node IDs raise** — `add_node` with an existing ID raises `Exception`
9. **Missing nodes raise `KeyError`** — `retract_node`, `show_node`, `what_if_*`, `challenge`, `defend` on nonexistent IDs
10. **Access control raises `PermissionError`** — `show_node` with `visible_to` that doesn't match the node's `access_tags`
11. **What-if operations never mutate** — verified by checking `get_status()` after the call
12. **Project isolation** — same node ID in different projects stores different data
13. **Challenge/defend is symmetric** — each challenge must be individually defended; the target stays OUT until all challenges are defeated

## 7. Error Handling

The tests verify three error categories:

- **`KeyError`** — raised for operations on nonexistent node IDs (`retract_node("nonexistent")`, `show_node("nonexistent")`, `what_if_retract("missing")`, `challenge("nonexistent", ...)`, `defend("nonexistent", ...)`, `add_justification("nonexistent", ...)`, `add_nogood(["nonexistent"])`)
- **`PermissionError`** — raised when `show_node` is called with a `visible_to` list that doesn't include any of the node's access tags
- **`Exception`** — raised on duplicate `add_node` calls (generic exception, not a specific subclass)
- **`ValueError`** — raised by `defend` when the `defense_id` collides with an existing node

Idempotent operations return empty change sets rather than raising: retracting an already-OUT node returns `{"changed": []}`, asserting an already-IN node does the same.

---

## Topics to Explore

- [file] `reasons_lib/pg.py` — The implementation under test; understanding how truth propagation is implemented in SQL (recursive queries vs. application-level BFS) is essential for debugging cascade failures
- [function] `reasons_lib/pg.py:what_if_retract` — Uses SAVEPOINTs or temp tables to simulate retraction without mutation; the no-side-effect guarantee is a key invariant
- [file] `tests/conftest.py` — Contains the `pg_api` fixture setup/teardown and `skip_no_pg` marker; understanding the isolation strategy (per-test project IDs, row cleanup) is critical for test reliability
- [general] `challenge-defend-protocol` — The dialectical protocol (challenge creates an outlist blocker, defend blocks the challenge) is a non-obvious architectural pattern; tracing it through `pg.py` reveals how it composes with standard TMS propagation
- [file] `reasons_lib/compact.py` — The `compact()` method tested in `TestCompact` generates budget-constrained markdown summaries with access filtering and summary-node elision; worth understanding for context-window management

## Beliefs

- `pg-api-what-if-never-mutates` — `what_if_retract` and `what_if_assert` are guaranteed read-only: they return cascade predictions without changing any node's truth value in the database
- `pg-retracted-pin-not-restored-by-antecedent` — When a node is explicitly retracted (pinned OUT), re-asserting its antecedent does not restore it; this distinguishes explicit retraction from cascade-induced OUT
- `pg-challenge-creates-outlist-blocker` — The `challenge()` method defeats a target by creating a new node that acts as an outlist blocker, using standard TMS outlist semantics rather than a special-case mechanism
- `pg-multi-tenancy-via-project-id` — PgApi isolates tenants by `project_id` column; the same node ID can exist independently across projects in the same database tables
- `pg-access-tags-inherited-from-antecedents` — When a derived node is created, it inherits the union of access tags from all its SL antecedent nodes

