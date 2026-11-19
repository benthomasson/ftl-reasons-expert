---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_pg.py

# `tests/test_pg.py` — PostgreSQL Backend Test Suite

## Purpose

This file is the **executable specification and regression suite for `reasons.pg.PgApi`**, the PostgreSQL-native storage backend of the “reasons” belief-network engine. It does not own production logic; its responsibility is to *pin down observable behavior* of the PG backend so that refactors, SQL schema changes, or propagation-algorithm tweaks don’t silently break semantics.

The tests act as:
- A contract document for every public method on `PgApi`.
- A cross-check that the PG backend implements the same **argumentation-network semantics** as the presumably in-memory backend (premises vs. derived nodes, IN/OUT truth, `SL` justifications, `unless` outlists, retraction cascades).
- A guard on multi-tenant isolation (`domain_id`) and access-tag filtering.

The whole module is gated by `pytestmark = [pytest.mark.pg, skip_no_pg]`, so these are skipped when no `DATABASE_URL`/PG is available; `pg_api` is a fixture from `tests/conftest.py` that hands each test a clean, isolated domain.

---

## Key Components

### Module preamble
- `pytestmark = [pytest.mark.pg, skip_no_pg]` — applies the `pg` marker and the skip condition to every test in the file. This is the *only* module-wide setup.
- `from tests.conftest import skip_no_pg` — pulls in the skip condition (likely `pytest.mark.skipif(not os.getenv("DATABASE_URL"), ...)`).

### Test classes (one per `PgApi` method family)

| Class | What it pins down |
|---|---|
| `TestAddNode` | `add_node(id, text, sl=None, unless=None, access_tags=...)` returns `{node_id, truth_value, type}`; type is `premise` when `sl` omitted, `derived` otherwise; duplicate `id` raises; access tags sorted/merged, derived nodes inherit ancestor tags. |
| `TestRefIntegrity` | `add_node`/`add_justification` reject `sl`/`unless` referencing missing nodes with `KeyError(match=id)`. Premise-only adds skip validation. |
| `TestRetractNode` | `retract_node(id)` returns `{changed, went_out}`, cascades; on already-OUT returns `changed == []`; accepts `reason=` stored in `metadata.retract_reason`; missing id → `KeyError`. |
| `TestAssertNode` | Inverse of retract: `{changed, went_in}`, cascades up the dependency graph; already-IN → empty. |
| `TestPropagation` | Diamond graphs, outlist blocking/unblocking, multiple justifications keeping a node IN when one path is OUT, and “retracted pin skipped” semantics. |
| `TestGetStatus` | `get_status(visible_to=None)` returns `{nodes, total, in_count}`, and `visible_to` filters by access tags. |
| `TestShowNode` | `show_node(id, visible_to=None)` returns `{id, truth_value, justifications, dependents, metadata, source}`; `KeyError` if missing; `PermissionError` if the caller’s `visible_to` misses a tag. |
| `TestSearch` | `search(q, format=...)` with `format` ∈ `None` (default text), `"compact"`, `"json"`, `"dict"`; hits text, ids, and one-hop neighbors; `dict` shape is `{results, count}` (plus `neighbors` when non-empty), empty is exactly `{"results": [], "count": 0}`. |
| `TestListNodes` | `list_nodes(status=, premises_only=, has_dependents=, namespace=)` returns `{count, nodes}`. |
| `TestListGated` | `list_gated()` returns `{blockers, blocker_count, gated_count}` — a reverse index of `unless` blockers → gated nodes with text. |
| `TestGetLog` | `get_log(last=N)` returns `{entries}` of `{action, ...}` records in chronological order. |
| `TestAddJustification` | Adding an `SL` justification flips `OUT → IN`, returns `{old_truth_value, new_truth_value}`; `KeyError` if target missing. |
| `TestNogood` | `add_nogood([...])` starts backtracking when all members are IN (`backtracked_to is not None`), is a no-op otherwise, `KeyError` on missing refs. |
| `TestFindCulprits` | `find_culprits([ids])` returns `{culprits}` ranked by entrenchment — nodes with `source=` rank higher, so unsourced is chosen first. |
| `TestExplainNode` | `explain_node(id)` returns `{steps}` with `reason`, `truth_value`, `antecedents`, `failed_antecedents`. |
| `TestTraceAssumptions` | `trace_assumptions(id)` returns `{premises}` — the transitive premise set (order may vary). |
| `TestMultiTenancy` | Two `PgApi` instances with different `domain_id` can share ids without colliding; cleanup deletes from `rms_propagation_log`, `rms_justifications`, `rms_nogoods`, `rms_network_meta`, `rms_nodes` filtered by `domain_id`. |
| `TestWhatIf` | `what_if_retract`/`what_if_assert` are *pure* — they predict `{already_out/in, total_affected, retracted, restored}` with `depth` fields but do not mutate. |
| `TestChallenge` | `challenge(id, reason, challenge_id=None)` creates a *premise* node `challenge-<id>` (or `challenge-<id>-2`, etc. on collision), retracts the target, sets `source="challenge"` and `metadata.challenge_target`; target records `metadata.challenges`. |
| `TestDefend` | `defend(target, challenge_id, reason, defense_id=None)` creates `defense-<challenge_id>`, retracts the challenge node (which re-propagates the target to IN). `defense_id` collision → `ValueError`. Multiple outstanding challenges keep the target OUT even after one defense. |
| `TestCompact` | `compact(budget=, visible_to=)` produces a budgeted markdown summary with sections `IN (active)`, `OUT (retracted)`, `Nogoods`; sorted by dependent-count descending; hides nogoods that reference private nodes; honors `metadata.summarizes` to elide covered nodes. |
| `TestExportNetwork` | `export_network(visible_to=)` returns the full network as `{nodes: {id: {...}}, nogoods: [...]}` — import-able round-trip format; filters private metadata keys (leading `_`). |
| `TestRemoveJustification` | `remove_justification(id, index)` returns `{remaining, removed, old_truth_value, new_truth_value, changed}`; refuses to remove the last justification (`ValueError`), refuses premise (`ValueError`), `IndexError` for bad index, `KeyError` for missing node. |
| `TestConvertToPremise` | `convert_to_premise(id)` strips justifications, keeps IN, propagates; returns `{old_justifications, truth_value, changed}`; no-op for a premise. |
| `TestGetBeliefSet` | `get_belief_set()` returns the list of IN ids. |
| `TestPropagate` | `propagate()` recomputes truth from scratch; useful after out-of-band DB writes (this test intentionally corrupts `rms_nodes.truth_value` via raw SQL). |
| `TestSupersede` | `supersede(old, new)` sets `metadata.superseded_by`/`supersedes`, retracts old, cascades; retracting `new` restores old (reversible via outlist-style semantics). |
| `TestSummarize` | `summarize(id, text, over=[...])` creates a derived node whose `SL` justification is `over`, tagged with `summarizes`/`summarized_by` metadata; `ValueError` on duplicate id, `KeyError` on missing reference. |
| `TestTraceAccessTags` | `trace_access_tags(id, visible_to=None)` unions access tags across the justification DAG; `KeyError` if missing; `PermissionError` if caller can’t see the node itself. |
| `TestEnsureNamespace` | Idempotent creation of `<ns>:active` premise with `metadata.role == "agent_premise"` and `metadata.agent == ns`. |
| `TestListNamespaces` | `list_namespaces()` returns `{namespaces: [{namespace, total_beliefs, in_beliefs, active}]}`; `active` reflects `<ns>:active` truth value. |
| `TestNamespaceAddNode` | Adding with `namespace=` prefixes the id, auto-creates `<ns>:active`, and adds the active premise as an implicit antecedent so retracting the “active” premise cascades the whole namespace. Already-prefixed ids are not double-prefixed. |
| `TestNamespaceAddJustification` | Justification calls with `namespace=` prefix `id`, `sl`, and `unless` consistently. |

### Raw DB access
Several tests reach past the API using `pg_api.conn` (a psycopg connection) and `pg_api.domain_id`:
- `TestPropagate` and `TestCompact.test_compact_summary_nodes` `UPDATE rms_nodes ... WHERE domain_id = %s`.
- `TestMultiTenancy` deletes directly from `rms_*` tables.

This confirms `PgApi` exposes `.conn` and `.domain_id`, and that all schema is scoped by `domain_id`.

---

## Patterns

- **One class per public method** — a documentation-like organization that makes the contract for each method scannable.
- **Fixture injection** — `pg_api` (from `conftest.py`) provides an isolated DB. Multi-tenancy test manually constructs a second `PgApi(os.environ["DATABASE_URL"], project2)` outside the fixture, then cleans it up in a `finally`.
- **Arrange-Act-Assert** throughout; assertions are on dict shape (`result["changed"]`) rather than opaque object state, so tests catch schema/response regressions.
- **Round-trip / idempotency checks** — `retract` then `assert`, `add_nogood` then defend, `supersede` then retract new — verifies reversibility.
- **Direct SQL mutation for edge cases** — when a scenario can only occur via corruption or external writes (`propagate` needs a stale truth value), the test bypasses the API.
- **Raising as assertion** via `pytest.raises(KeyError|PermissionError|ValueError|IndexError, match=...)`.
- **Storytelling** — many tests read like user actions: “challenge the premise, defend it, confirm the cascade is undone”.

---

## Dependencies

**Imports in the file**
- `json` — used to deserialize `search(..., format="json")` and to write metadata in `TestCompact.test_compact_summary_nodes`.
- `pytest` — fixtures, markers, `pytest.raises`.
- `tests.conftest.skip_no_pg` — the skip marker.
- `os`, `uuid`, `reasons.pg.PgApi` — imported inside `TestMultiTenancy.test_projects_isolated` only (deliberately local to keep other tests fast and skip-clean).

**Assumed from `conftest.py`**
- `pg_api` fixture returning a fresh `PgApi` bound to a unique `domain_id`.
- `skip_no_pg` marker object.

**Under test**
- `reasons.pg.PgApi` and the SQL schema `rms_nodes`, `rms_justifications`, `rms_nogoods`, `rms_network_meta`, `rms_propagation_log`.

**Nothing depends on this file** — it is a leaf in the dependency graph (test-only).

---

## Flow

Nothing non-standard; the *control flow of each test* is the standard `setup → act → assert`, with these recurring shapes:

1. **Propagation tests**: build graph → retract/assert → inspect the returned `changed/went_in/went_out` **and** re-query `show_node` (both the *side effect* and the *return value* are asserted).
2. **Search-format tests**: call `search` once per format (`None`, `"compact"`, `"json"`, `"dict"`) and assert type/markers per format.
3. **What-if tests**: call `what_if_*`, assert the *prediction*, then assert `get_status()`/`show_node()` to confirm **no mutation**.
4. **Namespace tests**: `add_node(..., namespace=ns)` implicitly creates `ns:active`, then subsequent `retract_node(ns:active)` should cascade everything in the namespace — this is the *coordinated whole-namespace retraction* mechanism.
5. **Multi-tenancy test**: builds a second `PgApi` with a fresh `uuid4()` project, exercises isolation, then `finally:` cleans up with raw `DELETE ... WHERE domain_id = %s` and `api2.close()`.

---

## Invariants

- **Referential integrity**: every `sl` and `unless` id must already exist; otherwise `KeyError` with the missing id in the message (regex `match="ghost"`, `"x.*y|y.*x"`).
- **Type classification**: `sl == None` ⇒ `type == "premise"`; otherwise `"derived"`. Only premises are directly retractable+restorable to their “natural” state.
- **Outlist semantics**: a derived node is OUT iff *all* its justifications fail; a justification with a non-empty `unless` succeeds only if every blocker is OUT.
- **Monotonic changed sets**: `retract_node` on already-OUT returns `changed == []`; `assert_node` on already-IN returns `changed == []`. Callers may therefore treat `changed` as “what actually moved”.
- **Multiple justifications**: a node stays IN as long as ≥1 justification succeeds (`test_multiple_justifications`, `test_remove_causes_truth_change`).
- **Access-tag inheritance**: derived nodes inherit the union of ancestor tags (`test_access_tag_inheritance`, `TestTraceAccessTags.test_chain_collects_all`); the returned list is sorted (`["aws", "billing"]`).
- **Namespace contract**:
  - `add_node(id, ..., namespace=ns)` yields id `<ns>:<id>`; if `id` already starts with `<ns>:`, no double prefix.
  - A namespaced add auto-creates `<ns>:active` (role `agent_premise`) and adds it as an implicit `sl` antecedent (visible in `explain_node`).
  - Therefore retracting `<ns>:active` cascades all namespaced beliefs.
- **Challenge/defend contract**:
  - Challenge node id defaults to `challenge-<target>`; on collision, suffix `-2`, `-3`, …
  - Challenge node has `source="challenge"`, `metadata.challenge_target=<target>`.
  - Target’s `metadata.challenges` records the challenge id.
  - Defend node id defaults to `defense-<challenge_id>`; has `metadata.defense_target=<challenge_id>`, `metadata.defends=<original target>`.
  - Defense retracts the *challenge* node (not the target directly), so target comes back IN only after all challenges are defended.
- **Justification invariants**:
  - A derived node must keep ≥1 justification (`remove_justification` refuses last → `ValueError`).
  - `remove_justification` on a premise → `ValueError` (premises have no justifications).
  - `convert_to_premise` strips all justifications and returns `old_justifications` count.
- **Nogood privacy**: nogoods that reference any node invisible to the caller are hidden from `compact(visible_to=...)`.
- **`export_network` filters private metadata**: keys beginning with `_` (e.g., `_retracted`) are not exported.
- **Multi-tenancy**: all state is scoped by `domain_id`; the same `id` in two `domain_id`s is fully independent, including nogoods and the propagation log.

---

## Error Handling

The suite treats the following exception classes as part of the API contract:

| Exception | Trigger in tests | Semantic meaning |
|---|---|---|
| `KeyError` | Missing node id in `show_node`, `retract_node`, `assert_node`, `what_if_*`, `trace_assumptions`, `trace_access_tags`, `convert_to_premise`, `supersede`, `challenge`, `defend`, `add_justification`, `remove_justification`; missing ids in `sl`/`unless` for `add_node`/`add_justification`; missing `over=` refs in `summarize`; missing member in `add_nogood`. | “You referenced an id that does not exist.” |
| `PermissionError` | `show_node` / `trace_access_tags` when `visible_to` misses a tag on the node or its ancestry. | Access-tag enforcement. |
| `ValueError` | `remove_justification` on the last justification or on a premise; `defend` with a colliding `defense_id`; `summarize` with a duplicate id. | Semantically invalid request. |
| `IndexError` | `remove_justification` with out-of-range index. | Index bounds on justification list. |
| `Exception` (broad) | `add_node` with a duplicate id (`pytest.raises(Exception)`). | The suite does not pin a specific class for duplicates, so a `KeyError`/`ValueError`/`IntegrityError` all satisfy it — worth tightening if you refactor. |

Errors are *never silently swallowed* in the API under test — every negative test asserts a raise. The suite does not test API methods that might intentionally return errors in a result dict (unlike `what_if_*`, which return status flags instead of raising).

---

## Topics to Explore

- [file] `tests/conftest.py` — Defines `pg_api` and `skip_no_pg`; understanding how each test gets an isolated DB and how `DATABASE_URL` gates the whole suite is prerequisite to running/debugging this file.
- [file] `reasons/pg.py` — The module under test; read alongside the class-by-class tests to see how the SQL, propagation, and access filtering actually work.
- [function] `reasons/pg.py:what_if_retract` — The pure-prediction engine; the `retracted`/`restored`/`depth`/`dependents` shape tested here is subtle (outlist “restore” branch interacts with normal retraction).
- [general] `argumentation-network-semantics` — IN/OUT truth, `SL` justifications, `unless` outlists, nogoods, entailment, and backtracking are the domain concepts all these tests encode; a formal model (e.g., defeasible logic) will make the diamond/diamond-restoration and “retracted pin skipped” tests obvious.
- [diff] `git log -- tests/test_pg.py` — This file’s test set is likely to evolve alongside `reasons/pg.py` feature additions (challenge/defend, namespaces, multi-tenancy); scanning history reveals which methods are newest and most likely to change.

## Beliefs

- `pytestmark-requires-database-url` — Every test in `tests/test_pg.py` is skipped unless `skip_no_pg` passes, which depends on a reachable PostgreSQL `DATABASE_URL`.
- `add-node-enforces-referential-integrity` — `PgApi.add_node` raises `KeyError` naming the missing id when any `sl` or `unless` id is absent, so phantom antecedents cannot enter the network.
- `namespace-add-node-implicitly-depends-on-ns-active` — `PgApi.add_node(..., namespace=ns)` auto-creates `<ns>:active` and adds it to the SL antecedents, so retracting `<ns>:active` cascades the whole namespace to OUT.
- `what-if-methods-do-not-mutate` — `what_if_retract` and `what_if_assert` return predictions but leave `get_status()`/`show_node()` unchanged, which the suite verifies explicitly in `test_what_if_retract_no_mutation` and `test_what_if_assert_no_mutation`.
- `defend-retracts-challenge-not-target` — `PgApi.defend(target, challenge_id, ...)` brings `target` back IN by retracting `challenge_id`, so with multiple outstanding challenges the target remains OUT until every challenge is defended (`TestDefend.test_defend_multiple_challenges`).
- `domain-id-is-the-multi-tenancy-key` — All persistence is partitioned by `domain_id`; two `PgApi` instances with distinct `domain_id`s can hold the same node id with independent content and no cross-contamination.
