---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_import_agent.py

# `tests/test_import_agent.py` — Test Suite for Multi-Agent Belief Import

## Purpose

This file is the behavioral specification for `reasons.api.import_agent` and the surrounding agent-namespacing feature. Its job is to pin down how snapshots of beliefs (in both **Markdown** and **JSON** form) get turned into a namespaced, survivable sub-network inside the shared belief store.

It is *not* testing the parser or storage layer directly — every assertion goes through the public `reasons.api` façade (`import_agent`, `show_node`, `retract_node`, `assert_node`, `what_if_retract`, `get_status`, `list_repos`) so the tests document the **external contract** of the import feature, plus a couple of regression seals for historical bugs (referenced as issue #16 in the code).

Responsibility, in one sentence: *guarantee that importing the same source under different agent names produces isolated, independently-retractable belief subgraphs that survive `recompute_all()` without resurrection.*

---

## Key Components

### Fixtures

| Fixture | Contract |
|---|---|
| `db(tmp_path)` | Creates a temp SQLite path, calls `api.init_db(db_path=...)`, returns the path string. Every test gets a fresh store. |
| `beliefs_file(tmp_path)` | Writes `SAMPLE_BELIEFS` to `beliefs.md` and `SAMPLE_NOGOODS` to a sibling `nogoods.md`, returns the path to the beliefs file. The importer discovers the nogood file by convention. |

### Constants

**`SAMPLE_BELIEFS`** — Markdown with three claims that exercise every status the importer must handle:
- `alpha-fact [IN] OBSERVATION` — a base fact (no dependencies)
- `beta-depends-alpha [IN] DERIVED` — depends on `alpha-fact`
- `gamma-stale [STALE] OBSERVATION` — a stale claim with a `Stale reason:` line

**`SAMPLE_NOGOODS`** — one nogood (`nogood-001`) that affects `alpha-fact, gamma-stale`. This checks that nogoods get remapped through the same namespace pass as beliefs.

### Test functions

The tests cluster into five themes.

**1. Basic import shape** — `test_import_agent_basic`, `test_import_agent_creates_premise`, `test_import_agent_namespaces_beliefs`

These pin the return dict of `import_agent`:
```python
{
  "agent": "test-agent",
  "prefix": "test-agent:",
  "active_node": "test-agent:active",
  "created_premise": bool,
  "claims_imported": int,
  "claims_retracted": int,
  "claims_skipped": int,
  "nogoods_imported": int,
}
```
And the metadata contract on `test-agent:active`:
```python
node["metadata"]["agent"] == "test-agent"
node["metadata"]["role"]  == "agent_premise"
```
Namespaced beliefs carry `metadata.original_id` so the source ID survives.

**2. Dependency remapping** — `test_import_agent_remaps_dependencies`

The structured justification on `test-agent:beta-depends-alpha` must satisfy:
```python
j["antecedents"] == [..., "test-agent:alpha-fact", ...]   # remapped
"test-agent:active"     not in j["antecedents"]            # issue-#16 fix
"test-agent:inactive"   in     j["outlist"]                # kill switch
```
This is the single most load-bearing assertion in the file: **the `active` premise is deliberately *not* an antecedent**, and every belief gets `{agent}:inactive` injected into its outlist as the retraction channel.

**3. Retraction / restore cascade** — `test_import_agent_retract_premise_cascades`, `..._actually_cascades`, `..._restore_premise_cascades`

Retracting `{agent}:active` must cascade to every belief. Because `active` is not an antecedent, the cascade is achieved by the "kill switch": when `active` is retracted, `{agent}:inactive` (which has `active` in its outlist) is computed IN, and every imported belief has `inactive` in its outlist, so they flip OUT. `assert_node("test-agent:active")` must reverse the whole thing.

The `what_if_retract` test asserts `total_affected == 3` — meaning `gamma-stale` is also IN at that moment, even though `test_import_agent_basic` counts it as `claims_retracted == 1`. This documents an important subtlety: **STALE claims are imported as retracted but get propagated back to IN during recompute** because their justification (empty antecedents, outlist = `[inactive]`) is satisfiable while `active` is IN.

**4. Idempotency, isolation, and formats** — `test_import_agent_skip_duplicates`, `test_import_agent_only_in`, `test_import_multiple_agents`, `test_import_agent_json`, `test_import_agent_json_outlist_blocks`, `test_import_agent_registers_repo`, `test_import_agent_nogoods`

- **Idempotency**: re-importing the same file returns `claims_imported == 0`, `claims_skipped == 3`, `created_premise is False`. Byte-identical source → no-op.
- **`only_in=True`**: skips any claim not marked `[IN]`, so `gamma-stale` never appears — `show_node("test-agent:gamma-stale")` raises `KeyError` *and* `claims_retracted == 0`.
- **Isolation**: importing two agents yields `total == 10` (`2 * (active + inactive + 3 beliefs)`). `retract_node("agent-a:active")` flips `agent-a:alpha-fact` OUT but leaves `agent-b:alpha-fact` IN.
- **JSON path**: the same namespacing/outlist/cascade invariants hold for the JSON export format, and `blocker-c` (which is `[OUT]` in source) keeps a single justification whose outlist includes `{agent}:inactive` — i.e. **retracted-on-import nodes are represented as a real OUT node gated by the inactive premise, not as "no justifications".**
- **Repo registration**: `import_agent` also writes the source file's parent directory into the repos table keyed by agent name (`api.list_repos()`).
- **Nogoods**: `result["nogoods_imported"] == 1`.

**5. Regression seals** — `test_import_agent_preserves_out_status`, `test_import_agent_supersession_preserved`, `test_retracted_belief_survives_propagate`, `test_retracted_json_belief_survives_propagate`, `test_out_with_satisfied_justifications_stays_out`

These exist because of two related bugs:

1. **"Resurrection"**: an `[OUT]` node in the source that had a satisfied justification (e.g. `derived-out` derived from `base-fact` which is `IN`) used to flip back to `IN` after import + recompute. The fix is that `[OUT]` nodes carry an extra "definitively retracted" marker (a `{agent}:inactive` gate) that survives `recompute_all()`.

2. **Issue #16**: the original design used `{agent}:active` as an antecedent of each imported claim. That made retracting a *single belief* effectively impossible — `active` was always IN, so recompute would always find "active ∈ antecedents ∧ active is IN" and put the belief back. Fix: remove `active` from antecedents and route retraction through `{agent}:inactive` in the outlist.

`test_retracted_belief_survives_propagate` and its JSON twin explicitly call the internals:
```python
from reasons.storage import Storage
store = Storage(db)
net = store.load()
net.recompute_all()
store.save(net)
store.close()
```
— because the resurrection bug only manifested on recompute, not on the direct API path.

---

## Patterns

- **Fresh-DB-per-test via `tmp_path` fixture** — no shared global state; tests are order-independent.
- **Public-API-only assertions** (except the two regression tests that reach into `Storage`/`recompute_all` to reproduce the exact failure mode).
- **Namespace-as-isolation**: the agent name is not a database column, it's a textual prefix (`{agent}:`). The store sees one flat graph; isolation is enforced by naming discipline + per-agent premise pairs.
- **Kill-switch via outlist**: rather than engineering a "disable subtree" primitive, the design piggybacks on the reasoning engine's existing `outlist` semantics (a justification fires iff all antecedents are `IN` **and** all outlist members are `OUT`). Importing a per-agent `inactive` premise whose justification is `SL: outlist=[active]` gives a single-wire off switch.
- **Regression tests named after bugs**: `test_retracted_belief_survives_propagate` embeds the issue number in the docstring ("core bug from issue #16") so future maintainers can find the historical context.
- **Return-dict contract**: `import_agent` returns a stats dict rather than raising on duplicates — makes idempotent sync the default UX.

---

## Dependencies

**Imports:**
- `os`, `tempfile` — imported but unused (dead imports; leftover from an earlier draft).
- `pytest` — fixtures and `pytest.raises(KeyError)`.
- `from reasons import api` — the primary contract under test.
- `from reasons.storage import Storage` — only in the two `_survives_propagate` regression tests, to invoke `recompute_all()` bypassing the API.
- `json`, `pathlib.Path` — file creation / path assertions inside individual tests.

**What depends on this file:**
- CI (`pyproject.toml` / `.github/workflows`) — invokes it as part of the suite.
- Nobody imports test modules. This file is a leaf.

**What it implies about the SUT:**
- `reasons/api.py` exports `import_agent`, `show_node`, `retract_node`, `assert_node`, `what_if_retract`, `get_status`, `list_repos`, `init_db`.
- `reasons/import_agent.py` exposes the implementation, and its file-discovery convention is "look for a sibling `nogoods.md` next to the beliefs file".
- `reasons/storage.py:Storage` has `.load()`, `.save(net)`, `.close()` and `Storage(db).load()` returns a network object with `.recompute_all()`.

---

## Flow

`import_agent(agent, path, db_path, only_in=False)` for each call:

1. **Open / init DB**, read the source (markdown or JSON — sniffed by content/extension).
2. **Create the pair of premises**:
   - `{agent}:active` — `IN`, `metadata.role == "agent_premise"`, `metadata.agent == agent`.
   - `{agent}:inactive` — a derived node whose justification is `SL: antecedents=[], outlist=[{agent}:active]`, i.e. `inactive ≡ ¬active`.
   - On re-import, `created_premise == False` because the premises already exist.
3. **For each source belief** (`[IN]` only if `only_in=True`):
   - Skip if `{agent}:{original_id}` already exists → `claims_skipped += 1`.
   - Otherwise create the node with `truth_value` from the source (`IN` or retracted for `STALE`/`OUT`).
   - **Rewrite** every `Depends on:` antecedent and every `Unless:` outlist entry to the `{agent}:` namespace.
   - **Do not** insert `{agent}:active` into `antecedents` (issue-#16 rule).
   - **Do** append `{agent}:inactive` to `outlist` (kill switch).
4. **For each nogood**: same namespace rewrite for `Affects:`.
5. **Register the agent → parent-dir mapping** in the repos table.
6. **Return the stats dict**.

When the caller later does `retract_node("{agent}:active")`, control flow is:
`active.truth_value = OUT` → `recompute_all()` → `inactive.evaluate()` becomes `IN` (its only outlist gate, `active`, is now `OUT`) → every imported belief, each of which lists `inactive` in its outlist, flips `OUT`. `assert_node("{agent}:active")` reverses the propagation.

---

## Invariants

- **`{agent}:active` appears in the antecedents of *zero* imported claims.** (`test_import_agent_remaps_dependencies`, `test_import_agent_json`). This is the explicit fix for issue #16.
- **`{agent}:inactive` appears in the outlist of every imported claim** (except the two agent premises themselves). This is the kill switch.
- **`inactive` is the dual of `active`**: when `active` is IN, `inactive` is OUT, and vice-versa. Retraction is achieved purely through this pair.
- **Namespacing is applied to *structured fields*, not only to text**: `Depends on:`, `Unless:`, and nogood `Affects:` all go through `{agent}:{original_id}`.
- **`[OUT]` in source ⇒ `OUT` forever.** Subsequent `recompute_all()` must not resurrect it (regression #16, `test_out_with_satisfied_justifications_stays_out`). The mechanism is that OUT nodes carry a justification with `inactive` in the outlist, so the "satisfied antecedents" path can never fire while `inactive` is OUT — and if `inactive` were IN, the node would still be OUT.
- **`original_id` metadata is preserved** for every imported belief, so the caller can map back to the source-file ID.
- **Idempotency**: importing the same file twice yields `claims_imported == 0` on the second call; nothing is overwritten.
- **Skipping is by *name collision***, not by content hash — a slightly edited source file still gets skipped on re-import (this is a deliberate conservative choice).
- **`only_in=True` is a strict filter** — no `STALE`/`OUT` claim is even created (a `KeyError` on `show_node` is the assertion).
- **Repo path recorded is the *resolved parent directory*** of the source file (`str(Path(beliefs_file).resolve().parent)`).
- **A STALE claim is imported as retracted but can become IN on recompute** if its justification is satisfiable — visible in `test_import_agent_retract_premise_cascades` (`total_affected == 3`).

---

## Error Handling

- **Duplicates do not raise.** The importer returns `claims_skipped` and `claims_imported == 0`.
- **`KeyError` is the contract for missing nodes** — used by `api.show_node`, verified by `with pytest.raises(KeyError)` in `test_import_agent_only_in`. Tests rely on this instead of a returned sentinel.
- **Failed imports are represented in the return dict**, not as exceptions. There is no negative count, no truthiness that would swallow errors — if the source file is malformed, `pytest` would surface the exception from `api.import_agent` directly since there's no `try/except` in this file.
- **The regression tests deliberately bypass nice APIs** (`Storage(db).load(); net.recompute_all(); store.save(net)`) because the bug they pin only exists in the recompute path — if the public API ever stops routing through that path, the tests should start failing rather than silently passing.
- **No teardown is required** — temp DBS are automatically disposed via `tmp_path`.

---

## Topics to Explore

- [file] `reasons/import_agent.py` — The production implementation whose contract this test suite defines; look for the markdown parser, the JSON path, the `{agent}:inactive` kill-switch wiring, and the `[OUT]`-preservation marker that survives `recompute_all()`.
- [function] `reasons/import_agent.py:import_agent` — Inspect the exact `outlist` injection order and how `only_in=True` short-circuits `[STALE]`/`[OUT]` claims before namespace rewriting happens.
- [general] `issue-16-resurrection-bug` — The regression this file seals; grep for `#16` in `reasons/` to see if other modules carry similar guards (e.g. `reasons/repair.py`, `reasons/sync_agent` if present in the future).
- [function] `reasons/api.py:init_db` — Records where the `repos` table lives that `test_import_agent_registers_repo` relies on; verify that the repos mapping is per-DB, not per-agent-name global.
- [file] `tests/test_sync_agent.py` — Likely pairs with this file; if it exists, it will show what happens when the *source file changes* between imports (this file only tests the unchanged-source idempotency case).

---

## Beliefs

- `active-precondition-is-not-an-antecedent` — For every imported belief, `{agent}:active` is excluded from `justifications[0].antecedents`; this is the explicit fix for the issue-#16 resurrection bug.
- `every-imported-belief-has-inactive-in-outlist` — Namespaced beliefs all carry `{agent}:inactive` in their outlist, which is what makes retracting `{agent}:active` cascade through every belief.
- `import-agent-registers-repo` — `import_agent` records the source file's resolved parent directory under `repos[agent]`, visible via `api.list_repos(db_path=...)["repos"]`.
- `out-source-status-survives-recompute` — A belief imported with source status `[OUT]` remains `OUT` even after `Storage(db).load().recompute_all()`, while `[STALE]` beliefs can be propagated back to `IN` on recompute.
- `duplicate-import-is-a-noop` — Re-importing an identical file returns `claims_imported == 0`, `claims_skipped == N`, and `created_premise == False`; no exceptions are raised for the collision.
- `import-agent-supports-markdown-and-json` — The public API `reasons.api.import_agent` accepts both the Markdown "Beliefs/Nogoods" export and the JSON network-export format, and both preserve the same namespacing/outlist invariants.
