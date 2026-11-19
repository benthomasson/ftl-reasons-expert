---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_api.py

# Explanation of `tests/test_api.py`

## Purpose

This file is the **behavioral contract test suite** for the public Python API in `reasons/api.py` (and, transitively, the reasoning engine behind it). It is the largest test file in the repository and exercises nearly every user-facing operation of the `reasons` belief-network system: database lifecycle, node CRUD, truth-value propagation, search (FTS, OR, regex, names-only), tag/source metadata, LLM-backed classification, clustering, semantic deduplication, and the full **proposal workflow** (propose → accept/reject/withdraw with drift and staling semantics).

Its role is not to test implementation details in isolation but to pin down the observable return shapes, side effects on the store, and error semantics of `reasons.api.*`. As a consequence, it doubles as executable documentation: for any function in `reasons/api.py`, you can find the expected dict keys, list ordering, and failure modes here.

## Key Components

### Fixture

- `db_path(tmp_path)` — creates `<tmp>/test_reasons.db`, calls `api.init_db(db_path=p)`, returns the path. Every test in the module takes `db_path` (or a scoped subclass fixture) so each test runs against an isolated, freshly initialized SQLite DB. `tmp_path` puts those DBs in per-test directories so parallel `pytest-xdist` runs are safe.

### Test Classes (grouped by API surface)

| Class | Exercises |
|---|---|
| `TestInitDb`, `TestAddNode`, `TestRetractNode`, `TestAssertNode`, `TestPropagate` | Core store lifecycle and 3-valued propagation (`IN`/`OUT`) |
| `TestGetStatus`, `TestShowNode`, `TestExplainNode` | Read-side inspection (`explain_node` returns an ordered `steps[]` trace) |
| `TestAddNogood`, `TestGetBeliefSet`, `TestGetLog`, `TestExportNetwork` | Constraint / witness / serialization APIs |
| `TestEndToEnd` | Chain a→b→c retract/assert round-trip (mirrors `test_network.py`) |
| `TestListNodesDepth`, `TestListGated`, `TestListNegative` | Filtered listings; `list_gated` reports `unless=` blockers; `list_negative` is LLM-backed |
| `TestFtsSearch` | All FTS behaviors: porter stemming, stop-word filtering, progressive relaxation, OR-pipes, `depth=` antecedent expansion |
| `TestSupersedeWithText`, `TestWhatIfSupersede` | Text replacement with / without mutation (`what_if_*` is a dry-run) |
| `TestSetMetadata`, `TestLifecycleTimestamps` | Metadata mutability; `created_at`/`updated_at`/`retracted_at`/`verified_at`/`reviewed_at` semantics |
| `TestListClusters`, `TestDeduplicateSemantic` | Clustering passthrough + semantic dedup (skipped without optional deps) |
| `TestProposeRetraction`, `…Supersession`, `…Addition`, `TestProposeNogood` | Proposal creation, ID shapes (`prop-<id>-<action>-<n>`), impact estimation, auto-staling of older pendings |
| `TestAcceptProposal`, `TestRejectProposal`, `TestWithdrawProposal` | Proposal resolution; accept applies the action; target-drift detection marks the proposal `stale` instead of applying |
| `TestProposalTags`, `TestStoreLlmProposals` | Proposal tag filtering/sorting; batch ingest of LLM JSON proposals (update→supersede, skips with reasons) |
| `TestOrSearch`, `TestRegexSearch`, `TestNamesOnlyFormat`, `TestSearchPagination`, `TestStatusPagination` | Query-language variants and pagination (`limit`/`offset`) |
| `TestNodeTags`, `TestNodeSources`, `TestExportImportTagsSources`, `TestTagFiltering` | `key:value` tag namespace, sources, export/import round-trips, tag-filtered list/search |

### Constants / module-level

- `HAS_CLUSTER_DEPS` (from `reasons.cluster`) with a graceful `ImportError` fallback to `False`.
- `skip_no_cluster` — `pytest.mark.skipif` used only on `TestDeduplicateSemantic`.
- No production code is defined here; the file is pure test.

### Representative contracts asserted

- `api.init_db(db_path)` → `{"created": True}`; raises `FileExistsError` on an existing path; `force=True` overwrites.
- `api.add_node(id, text, sl="a,b", unless="x", label="…", source=…, access_tags=[…])` → `{node_id, truth_value, type}`, where `type` is `"premise"`, `"SL"`, etc.
- `api.retract_node(id)` → `{"changed": [ids…]}` (empty if already OUT); cascades to dependents; raises `KeyError` if missing.
- `api.assert_node(id)` → `{"changed": [...]}` mirror image of retract.
- `api.propagate()` → repairs the store's truth values even when storage was corrupted externally (see `TestPropagate.test_with_changes`, which directly writes a wrong `truth_value` via `Storage` before calling).
- `api.explain_node(id)` → `{"steps": [{"node", "reason"}, …]}`, with `reason` `"premise"`, `"circular …"` for genuine cycles, and never `"circular"` for diamonds.
- `api.show_node(id)` → includes `metadata`, `tags`, `sources`, all five timestamp fields.
- `api.search(query, format=…, depth=…, limit=…, offset=…, regex=…, tag=…)` — `format` ∈ `{"names-only", "compact", …}`; `|` is treated as OR at the query layer; `regex=True` compiles Python `re` and raises `ValueError` on bad patterns.
- `api.list_gated()` → `{"blockers": {blocker_id: {"text", "gated": [{"id"}]}}, "blocker_count", "gated_count"}` — only active (IN) blockers whose gated concluions are not superseded.
- `api.list_negative(visible_to=[…])` → LLM classification of keyword-matched candidates; returns `{"negative", "count", "candidates", "total"}`; batching behavior is pinned (see below).
- `api.propose_*(...)` / `api.accept_proposal` / `api.reject_proposal` / `api.withdraw_proposal` — full proposal state machine with `status ∈ {"pending","accepted","rejected","withdrawn","stale"}`.
- `api.add_tags` / `remove_tags` / `get_tags` — tags are strings with a `key:value` shape; `access_tags=["internal"]` at `add_node` time is migrated to `access:internal`.
- `api.add_source(node_id, source_ref, source_type=…)` → `{"node_id", "source_id"}`; deduped against `add_node(..., source=…)`.

## Patterns

1. **Class-as-fixture-grouping.** Each test class covers exactly one public function (or tight cluster of functions). The shared `db_path` fixture makes each class a small self-contained "unit of the API."

2. **Two-layer black-box testing.** The tests avoid mocking the API itself; they only mock external boundaries (`reasons.llm.invoke_model`, `reasons.cluster.list_clusters`) and only patch internals in *two* white-box cases (`_fts_search`, `_fts_query`) where the test asserts an algorithmic invariant (query-call-count bound). This is a deliberate bias: surface behavior against the real store, mocking only the cheap/dangerous parts.

3. **Round-trip contract testing.** `TestExportImportTagsSources`, `TestLifecycleTimestamps.test_import_roundtrips_timestamps`, and `TestLifecycleTimestamps.test_verified_at_preserved_through_export_import` exercise `export_network` → JSON on disk → `import_json` and assert each field (tags, sources, `created_at`, `retracted_at`, `verified_at`) survives verbatim.

4. **Dry-run / mutation pairs.** `what_if_supersede` is tested side-by-side with `supersede_with_text`, and `propose_*` is tested to *not* mutate while `accept_proposal` is tested to apply. The pattern `assert node["truth_value"] == "IN"` after a dry-run is a deliberate "no mutation" assertion.

5. **Drift / stale as first-class state.** `test_drift_target_already_out`, `test_drift_target_deleted`, `test_drift_add_node_already_exists`, `test_drift_supersede_new_id_collision` all follow the shape: propose → mutate the store behind the proposal's back → `accept_proposal` returns `{"applied": False, "status": "stale"}`. This pins the optimistic-concurrency contract.

6. **Auto-staling rules.** `test_auto_stales_older_pending` in each `TestPropose*` class asserts that a second pending proposal with the **same** `(action, target)` stales the first, while `test_different_actions_dont_stale_each_other` / `test_different_actions_independent` assert cross-action proposals remain independent.

7. **Skip-guard for optional heavy deps.** `HAS_CLUSTER_DEPS` + `skip_no_cluster` keep CI green when `sentence-transformers` and `scikit-learn` aren't installed, without marking the entire module skip.

8. **Isolation via `Storage` hacks.** A few tests bypass the API to force impossible intermediate states (`TestPropagate.test_with_changes`, `test_verified_at_preserved_through_export_import`, `TestProposeNogood.test_accept_nogood_missing_node`) by loading `Storage(db_path)` directly and editing `net.nodes[...]`. This is used specifically to test the repair / drift code paths that a well-behaved caller can't reach.

## Dependencies

**Imports (this file → elsewhere):**

- `pytest` — fixtures, `pytest.raises`, `pytest.mark.skipif`.
- `unittest.mock.patch` / `patch as mock_patch` — patching `reasons.llm.invoke_model`, `reasons.cluster.list_clusters`, and (white-box) `reasons.api._fts_query`.
- `reasons.api` — the module under test; also exposes private helpers `_fts_search` and `_fts_query` which the tests import directly.
- `reasons.storage.Storage` — reached into directly in three tests to corrupt/set state the public API doesn't allow.
- `reasons.cluster.HAS_CLUSTER_DEPS` — optional-dependency flag.
- Stdlib: `json`, `datetime`.

**Imported_by (elsewhere → this file):** none. Test files are leaves; only `pytest` collection traverses them. `tests/conftest.py` may supply shared fixtures, but this file only depends on the built-in `tmp_path`.

**Transitive production surface pinned here:** `reasons/api.py` (the functions listed above), `reasons/storage.py` (`Storage.load`/`save`/`close`), `reasons/llm.py` (`invoke_model`), `reasons/cluster.py` (`list_clusters`, `HAS_CLUSTER_DEPS`).

## Flow

Each test follows the same skeleton:

1. `tmp_path` is provided by pytest → `db_path` fixture calls `api.init_db(db_path=p)`.
2. Test body invokes one or more `api.*` mutating functions (`add_node`, `retract_node`, `propose_*`, `accept_proposal`, …).
3. Assertions run against **two orthogonal surfaces**:
   - Return value of the API call (`result["changed"]`, `result["status"]`, `result["count"]`, shape of `steps`).
   - Follow-up read of persisted state via `show_node` / `get_status` / `export_network` / `get_tags` / `get_belief_set`.
4. LLM/cluster tests patch their respective module's entry point (`reasons.llm.invoke_model`, `reasons.cluster.list_clusters`) so no real model is invoked; `mock_llm.assert_not_called()` is used to pin short-circuit paths (empty DB, no keyword hits, `skip_llm=True`).
5. Tests run in process; DB is torn down by pytest with `tmp_path`.

**Notable control-flow tests to understand:**

- `TestFtsSearch.test_long_query_does_not_explode` — patches `reasons.api._fts_query` with a counting wrapper and asserts `call_count <= 51` for a 20-term query. This documents that progressive relaxation is bounded (roughly O(n) fallback attempts with a hard cap), so a malicious query can't do arbitrary amounts of SQLite work.
- `TestFtsSearch.test_depth_0/1/2_*` — pins that `search(..., depth=N)` walks up to N antecedent hops; depth 0 = only the matching node; depth 1 = direct `sl=` parents; depth 2 = grandparents.
- `TestListNegative.test_batching_large_set` — 120 candidate nodes → exactly **3** LLM calls (i.e., batch size ~50). The mock returns different JSON per call to prove aggregation across batches.
- `TestExplainNode.test_explain_circular_dependency` — artificially creates a cycle (`x` ← `y` ← `x` via `add_justification`) and asserts a step reason containing `"circular"`; the diamond test asserts no false-positive circularity for a `root → {left, right} → top` DAG.
- `TestProposeRetraction.test_proposal_ids_increment` — pins the ID format: `prop-{target_id}-{action}-{sequence}`.

## Invariants

The file enforces these rules of `reasons.api`:

1. **DB lifecycle.** `init_db` never silently overwrites; without `force=True` on a non-empty path it raises `FileExistsError`.
2. **Node identity.** Duplicate `add_node(id)` raises `ValueError`. `show_node`/`retract_node`/`assert_node`/`set_metadata`/`add_source` on a missing id raise `KeyError`.
3. **Three-valued discipline.** `retract_node` is idempotent in effect: retracting an already-OUT node returns `{"changed": []}` and does not raise. `assert_node` on an already-IN node returns `{"changed": []}`.
4. **Cascade completeness.** Retracting a premise cascades to *all* transitive dependents (a→b→c all go OUT). Assertion is symmetric.
5. **Cascade ≠ direct retraction.** `retracted_at` is set exactly on the node the user retracted; cascade-affected dependents get `truth_value == "OUT"` but **empty** `retracted_at` (`test_cascade_does_not_set_retracted_at_on_dependents`). Asserting the node clears its own `retracted_at`.
6. **Proposal state machine.** `status` begins as `"pending"`; can only transition to `accepted`/`rejected`/`withdrawn`/`stale`. Accepting or rejecting a non-`pending` proposal raises `ValueError("… not pending")`.
7. **Proposals do not mutate the store until accepted.** Every `TestPropose*` has a `test_no_*_change` or `test_no_truth_change` asserting nothing moved.
8. **Optimistic concurrency on accept.** If the target (or the new id, or the nogood's node set) changed between propose and accept, `accept_proposal` returns `{"applied": False, "status": "stale"}` and marks the proposal `stale` — no partial mutation.
9. **Auto-staling is scoped to (action, target).** Re-proposing the same action on the same target stales earlier pendings; a different action (e.g. retract then supersede) does not.
10. **Query language.** `a|b` is a disjunction; `regex=True` uses Python `re` with case-insensitivity; invalid patterns raise `ValueError`. FTS goes through porter stemming and progressive relaxation but the number of `_fts_query` calls for a single search is bounded (≤51 in the tested scenario).
11. **Tag namespacing.** `access_tags=[...]` at `add_node` time are migrated into `access:<value>` tags (`TestNodeTags.test_access_tags_migrated_on_add`). `list_nodes(tag={"topic": ["networking", "auth"]})` is OR within a key and AND across keys.
12. **Metadata survival.** Tags and sources survive `retract`/`assert` and full export→import round-trips with sub-fields (`source_type`, `label`) preserved.
13. **No-op passthroughs.** `list_gated` on a DB with no `unless=` beliefs returns `{"blockers": {}, "gated_count": 0}`; `list_negative` on an empty DB or a DB with no keyword candidates never calls the LLM.

## Error Handling

The suite is as much about what `reasons.api` *raises* as about what it returns:

- **Domain errors → exceptions at the boundary:**
  - `FileExistsError` — `init_db` on existing DB.
  - `ValueError` — duplicate node id, accept/reject/withdraw of non-`pending` proposal, bad regex, `propose_nogood` with <2 nodes or a missing node (via `KeyError` with message "not found"), `propose_*` on already-OUT target, `propose_addition` with existing id, `propose_supersession` with colliding `new_id`.
  - `KeyError` — any read/update against a non-existent node/proposal.
  - `FileNotFoundError` — propagates from `list_negative` when the `claude` CLI is missing (see `test_claude_not_found_propagates`): the API does **not** swallow this — setup failures must surface.

- **Malformed LLM output is swallowed at the feature boundary, not at the caller.** `TestListNegative.test_malformed_llm_response` (prose instead of JSON) and `test_prose_with_brackets_before_json` show that `list_negative` extracts the first valid JSON array from surrounding prose and returns `count == 0` (or 0 for unparseable) rather than raising. `test_llm_returns_unknown_ids` shows unknown ids are silently dropped — the API returns only ids that were actually in the candidate set.

- **Drift itself is a non-error result.** `accept_proposal` on a drifted target does *not* raise — it returns `{"applied": False, "status": "stale"}`, which callers can branch on. This is a deliberate "soft failure" contract, different from the "hard errors" that missing records raise.

- **`_fts_search` bounds.** Long queries silently fall back to fewer terms (progressive relaxation) rather than raising; the caller never sees the internal relaxation. Only the white-box test guards the bound.

---

## Topics to Explore

- [file] `reasons/api.py` — The module under test; skim to see how each `Test*` class maps to a public function and what kwargs it accepts.
- [function] `reasons/api.py:_fts_search` — Progressive relaxation algorithm: term stripping, stop-word filtering, OR handling, and the bounded fallback count asserted by `test_long_query_does_not_explode`.
- [file] `reasons/storage.py` — Persistence layer (`Storage.load/save`, cascade logic, `retracted_at` semantics). Several tests reach into it directly to force intermediate states.
- [file] `reasons/propose_update.py` — The proposal state machine: IDs, staling rules, drift detection on `accept_proposal`, and the `update`→`supersede` translation performed by `store_llm_proposals`.
- [general] `proposal-drift-and-staling` — The optimistic-concurrency model this suite pins: `(action, target)` staling + drift-to-stale on accept, and how callers are expected to react.

---

## Beliefs

- `retract-cascades-to-dependents` — Retracting a node sets every transitive `sl=` dependent to `OUT`, and asserting it back restores the whole subtree symmetrically.
- `init-db-refuses-existing-without-force` — `api.init_db` raises `FileExistsError` on an existing database path unless `force=True` is passed.
- `accept-proposal-drift-yields-stale-not-error` — When the target state changed between proposal and acceptance, `accept_proposal` returns `{"applied": False, "status": "stale"}` and does not mutate the store, rather than raising.
- `fts-query-calls-are-bounded` — A single `_fts_search` call for a 20-term query invokes `_fts_query` at most 51 times, evidencing a hard cap on progressive-relaxation fallback attempts.
- `cascade-does-not-set-retracted-at` — `retracted_at` is populated only on nodes the user directly retracted; cascade-affected dependents end up `OUT` with an empty `retracted_at`.
- `access-tags-migrate-to-tag-namespace` — Passing `access_tags=["internal"]` to `api.add_node` results in a stored tag `access:internal`, not a raw `access_tags` field.
