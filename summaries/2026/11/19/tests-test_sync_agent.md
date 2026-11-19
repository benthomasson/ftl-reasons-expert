---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_sync_agent.py

# Explanation: `tests/test_sync_agent.py`

## Purpose

This file is the **behavioral contract for `api.sync_agent`** — the code path that keeps an agent's locally-stored beliefs in sync with an external source file (Markdown or JSON) *after* an initial `import_agent`. It verifies one specific policy that the sync operation owns:

> **Remote wins.** When the remote file and the local store disagree, the remote representation is authoritative — including for cases where the user locally retracted a node, changed dependency edges, or wants a previously retracted node back.

The suite also encodes regression expectations discovered during development, most notably that syncing a belief must not break the agent-revocation cascade (see `TestSyncAgentRevocation`).

It covers two serialization formats:
- **Markdown** (default agent-export format, with `### name [TRUTH] TYPE` headers and `- Depends on:` bullets).
- **JSON** (`{"nodes": {...}, "nogoods": [...]}` network dumps).

## Key Components

### Module-level constant

- **`INITIAL_BELIEFS`** — Canonical fixture text with three beliefs that exercise three orthogonal cases:
  - `alpha-fact [IN] OBSERVATION` — an unconditional premise.
  - `beta-depends-alpha [IN] DERIVED` with `- Depends on: alpha-fact` — demonstrates the antecedent edge.
  - `gamma-stale [STALE] OBSERVATION` — **STALE is imported as `OUT`**, so it already counts as a retraction on initial import. Many tests reuse this.

### Fixtures

- **`db(tmp_path)`** — Builds an isolated SQLite DB per test and calls `api.init_db`. Returns just the path string.
- **`initial_import(db, tmp_path)`** — Writes `INITIAL_BELIEFS` to `beliefs.md`, runs `api.import_agent("test-agent", …)`, and returns `(db_path, tmp_path)`. Every test that needs "prior state" starts here, so `test-agent` is the conventional agent under test.

### Test classes (contract clusters)

| Class | What it pins down |
|---|---|
| `TestSyncNoChanges` | Identical file → `beliefs_added == 0`, `beliefs_removed == 0`, `beliefs_retracted == 1` (that's `gamma-stale`, still OUT). |
| `TestSyncNewBeliefs` | Remote-only node → `beliefs_added == 1`; `show_node` returns `truth_value == "IN"` and `metadata["agent"] == "test-agent"`. |
| `TestSyncRemovedBeliefs` | Node present locally, absent remotely → `beliefs_removed == 1`; the node becomes `OUT` (retracted, not deleted). |
| `TestSyncUpdatedText` | Splicing in a Unicode char (`Α`) → `beliefs_updated >= 1`, `"Α" in node["text"]`. |
| `TestSyncTruthValueChanges` | IN→OUT propagates through to dependent (`beta`); STALE→IN clears retraction. |
| `TestSyncRetractedCleared` | Local `retract_node` followed by sync against unchanged remote → node goes back to `IN`; dependent resumes `IN`. **This is the "remote wins" test.** |
| `TestSyncUpdatedDependencies` | Dependency edge swap (`alpha-fact` → `gamma-stale`); asserts `antecedents` reflect new edge and old one is gone. |
| `TestSyncJson` | The three main scenarios replay through the JSON path, proving `sync_agent` autodetects format. |
| `TestSyncCountingAccuracy` | `beliefs_updated == 0` for a no-op; **justification-only** change (dep swap, same text) still yields `beliefs_updated >= 1`. |
| `TestSyncFirstTime` | Calling `sync_agent` on a fresh agent works as an initial import and returns `created_premise: True`. |
| `TestSyncIdempotency` | A second sync after a removal must not re-count `beliefs_removed`; a second identical sync is fully a no-op. |
| `TestSyncAgentRevocation` | **Regression**: after sync, justifications still contain `"test-agent:inactive"` in their `outlist`; retracting `test-agent:active` still cascades every belief `OUT`. |
| `TestSyncOutWithJustifications` | JSON: IN→OUT preserves justifications; OUT→IN clears `_retracted`; `OUT` with justifications is idempotent. |
| `TestSyncRegistersRepo` | After sync, `api.list_repos()["repos"]["test-agent"] == str(Path(str(p)).resolve().parent)`. |

## Patterns

1. **Fixture-scoped isolation.** Every test gets a fresh `tmp_path` + SQLite DB. No global state.
2. **Data-driven variants via `.replace()`.** Tests like `test_sync_in_to_out` build a variant by string substitution, keeping the fixture readable and the delta minimal.
3. **Return-value contract.** `sync_agent` is treated as returning a dict with named counters (`beliefs_added`, `beliefs_removed`, `beliefs_updated`, `beliefs_retracted`, `beliefs_unchanged`, `created_premise`). Tests pin these counters exactly or with `>=`.
4. **Side-effect verification.** Counts come from the return value, but state is checked via `api.show_node(<id>)` → `truth_value`, `text`, `metadata`, `justifications`.
5. **Regression test naming.** Tests like `test_synced_beliefs_still_cascade_on_agent_revocation` include docstrings that describe the specific bug ("assert_node during sync must not detach beliefs from their justifications"). Treat any failing test here as a bug fix, not a spec change.
6. **Agent-prefixed node IDs.** All assertions use the `<agent>:<name>` form (`test-agent:alpha-fact`), confirming the agent namespace is part of the storage contract.
7. **Markdown↔JSON symmetry.** `TestSyncJson` mostly mirrors `TestSyncNoChanges / Removed / Cleared` classes, verifying format-agnostic behavior.

## Dependencies

**Imports (what this file needs):**

- `json` — constructing JSON-format payloads for the JSON sync tests.
- `pytest` — fixtures (`tmp_path`) and test discovery.
- `from reasons import api` — the module under test.
- `pathlib.Path` — resolving the repo path in `test_sync_registers_repo`.

**API surface exercised:**
- `api.init_db(db_path=…)`
- `api.import_agent(agent, path, db_path=…)` — the "prior state" seeder.
- `api.sync_agent(agent, path, db_path=…)` — **the function under test**.
- `api.show_node(node_id, db_path=…)` — read-only inspections.
- `api.retract_node(node_id, db_path=…)` — mutates truth value → `OUT`.
- `api.list_repos(db_path=…)["repos"]` — registry inspection.

**Reverse deps (who imports this file):** nobody — it lives in `tests/` and is collected by pytest. Its only consumers are the test runner and CI (`.github/workflows/`).

**Implementation targeted transitively:** `reasons/api.py` and `reasons/import_agent.py` at minimum (the sync logic itself lives in one of these — likely `import_agent.py`, since it produces the file authoring format). `reasons/network.py` is involved in the cascade-tests through the `outlist`/`inactive_id` mechanism.

## Flow (what a single sync does, inferred from assertions)

```
sync_agent(agent, path, db_path)
 ├─ detect format (JSON object vs Markdown headers)
 ├─ if agent's "active" premise doesn't exist:
 │     create it  →  result["created_premise"] = True
 ├─ parse remote into {node_id → {text, truth, justifications}}
 ├─ for each remote node:
 │     ├─ absent locally            → add (truth value honored)   → beliefs_added += 1
 │     ├─ present, text/j-deps same →                             → beliefs_unchanged += 1
 │     ├─ present, text or deps differ → update                  → beliefs_updated += 1
 │     └─ present, truth differs:
 │           IN→OUT   → retract; dependent cascades              → beliefs_updated += 1
 │           OUT/STALE→IN → clear _retracted; resurrect          → beliefs_updated += 1
 ├─ for each local node not in remote:
 │     ├─ if already OUT or STALE   → skip (idempotency)
 │     └─ else                      → retract                    → beliefs_removed += 1
 ├─ register the file's directory under agent                    → list_repos
 └─ return summary dict
```

Key subtlety confirmed by `TestSyncNoChanges`: `gamma-stale` is imported as `OUT` but keeps a `[STALE]` marker; a subsequent unchanged sync reports `beliefs_retracted == 1` (it's still OUT) but `beliefs_removed == 0` (it wasn't in the "just removed" bucket). That is, **retracted ≠ removed** — `removed` counts *transitions* from IN→OUT caused by the sync, while `retracted` counts nodes that ended up OUT in the sync pass.

## Invariants Enforced

1. **Remote wins over local retraction.** If the remote says `IN`, the local node is resurrected, and its dependents come back too (`TestSyncRetractedCleared`).
2. **Removal is retraction, not deletion.** A node removed from remote is marked `OUT` but remains queryable via `show_node` (`test_sync_retracts_removed_belief`).
3. **Cascade preservation.** Whatever sync does internally to assert nodes must **not** strip `test-agent:inactive` from a justification's `outlist`. If it does, a later `retract_node("test-agent:active")` will no longer cascade (`TestSyncAgentRevocation`). This is the single most load-bearing invariant in the file.
4. **Idempotency.** Running sync twice against the same file must produce `0` for `beliefs_added`, `beliefs_removed`, `beliefs_updated` on the second run. The first run's removed-retractions must not be double-counted.
5. **Truth-value transitions imply `_retracted` state changes.** OUT→IN must clear `_retracted` (see `test_sync_out_to_in_clears_retracted`); IN→OUT must preserve the new `justifications` list and force `OUT` regardless of network derivation.
6. **Counter granularity.** Justification-only changes count as `beliefs_updated`, not `beliefs_unchanged` — the counter is meant to reflect "did anything semantically change".
7. **Repo registry.** Sync (like import) records `agent → parent-directory-of-source-file`. `Path(...).resolve()` is required because the assertion compares against a resolved absolute path.
8. **STALE maps to OUT.** The markdown `[STALE]` tag translates to `OUT` truth value on the stored node. Sync must treat an OUT node as retracted for the purposes of the "should I re-retract it?" check.

## Error Handling

The test file does **not** exercise error paths — there are no `pytest.raises` blocks. That means:

- **Valid inputs only.** The suite presumes the source file is well-formed and the DB is reachable.
- **Bad input, missing agent, malformed JSON, corrupt DB, or unknown `agent` argument are all undefined by this file.** If you need those contracts, look for a companion test (none currently enumerated in the repo tree besides `test_import_agent.py` / `test_import_json.py`, which exercise the import side).
- **No exception should escape normal-path sync.** Every test calls `sync_agent` bare and then asserts on the returned dict, so the contract is "return a summary, do not raise".

This is worth flagging to a maintainer: error-path coverage for `sync_agent` is a gap in this file.

---

## Topics to Explore

- [file] `reasons/import_agent.py` — Likely home of the parsing + sync logic; start here to see how format detection and delta computation are actually implemented.
- [function] `reasons/api.py:sync_agent` — The concrete entry point these tests target; check its return-dict assembly to see exactly which keys are guaranteed.
- [file] `reasons/network.py` — Understand `_retracted`, `assert_node`, `retract_node`, and the `outlist`/`inactive_id` cascade the regression tests depend on.
- [general] `agent-revocation-cascade` — The `inactive` premise + justification-outlist mechanism; it is the hardest invariant to maintain through sync and is the reason `TestSyncAgentRevocation` exists.
- [general] `sync-error-paths` — Nothing in this file checks invalid inputs, missing files, or DB failures; look for or add tests covering those, and inspect `sync_agent`'s exception handling.

## Beliefs

- `sync-agent-implements-remote-wins` — When a belief exists locally but is retracted, `sync_agent` reasserts it to IN if the remote source declares it IN; a subsequent sync is idempotent.
- `sync-agent-preserves-agent-revocation-cascade` — Sync must leave `agent:inactive` in each justification's `outlist`; otherwise retracting `agent:active` will not cascade beliefs to OUT.
- `sync-requires-remote-vs-local-format-detection` — The same `sync_agent` function handles both Markdown (`### name [TRUTH] TYPE`) and JSON (`{"nodes": ...}`) sources without a format argument.
- `sync-removal-is-retraction-not-deletion` — Nodes removed from the remote source end up with `truth_value == "OUT"` and remain queryable via `show_node`.
- `sync-idempotency-requires-counting-transitions` — `beliefs_removed` counts *transitions* IN→OUT during the sync pass; re-syncing an already-removed node yields `0`, not a repeat increment.
- `stale-tag-maps-to-out` — A `[STALE]` markdown tag imports as `OUT` and is subsequently counted in `beliefs_retracted`, not `beliefs_removed`, on an unchanged re-sync.
- `justification-changes-count-as-updates` — A dependency-edge change with identical text yields `beliefs_updated >= 1`, not `beliefs_unchanged`.
