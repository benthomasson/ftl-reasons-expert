# File: tests/test_sync_agent.py

**Date:** 2026-05-08
**Time:** 14:12

# `tests/test_sync_agent.py`

## Purpose

This file is the specification-as-tests for `api.sync_agent` — the operation that re-imports an agent's beliefs from a remote file (markdown or JSON) into the local TMS database, applying a **remote-wins** conflict resolution strategy. It covers the full lifecycle: initial sync, additions, removals, text updates, truth-value changes, dependency rewiring, idempotency, and interaction with the agent revocation mechanism.

It is the only test file for this feature and effectively serves as the contract definition for `sync_agent`'s return value and side effects.

## Key Components

### `INITIAL_BELIEFS` (constant)

A markdown string defining three beliefs used as the baseline state for most tests:

| ID | Status | Type | Role in tests |
|---|---|---|---|
| `alpha-fact` | IN | OBSERVATION | Stable premise; dependency target |
| `beta-depends-alpha` | IN | DERIVED | Tests cascade behavior (depends on alpha) |
| `gamma-stale` | STALE | OBSERVATION | Tests STALE→OUT mapping and resurrection |

This constant is the "remote" document that most tests start from.

### Fixtures

- **`db`** — Creates a fresh SQLite database in `tmp_path` via `api.init_db`. Every test gets an isolated DB.
- **`initial_import`** — Writes `INITIAL_BELIEFS` to disk, runs `api.import_agent("test-agent", ...)`, and returns `(db_path, tmp_path)`. This is the "world before sync" for most test classes.

### Test Classes (11 classes, ~20 tests)

| Class | What it proves |
|---|---|
| `TestSyncNoChanges` | Syncing identical data is a no-op (counts are zero except pre-existing retractions) |
| `TestSyncNewBeliefs` | New beliefs in remote are added and namespaced (`test-agent:delta-new`) |
| `TestSyncRemovedBeliefs` | Beliefs absent from remote are retracted (set OUT) locally |
| `TestSyncUpdatedText` | Changed description text propagates; `beliefs_updated` counter increments |
| `TestSyncTruthValueChanges` | IN→OUT cascades to dependents; STALE→IN resurrects |
| `TestSyncRetractedCleared` | Remote IN overrides a local retraction ("remote wins"); dependents restored |
| `TestSyncUpdatedDependencies` | Justification antecedents are rewired when remote changes `Depends on:` |
| `TestSyncJson` | JSON format (`network.json`) works for add, update, remove, and retraction-clearing |
| `TestSyncCountingAccuracy` | `beliefs_unchanged` and `beliefs_updated` counts are precise; justification-only changes count as updates |
| `TestSyncFirstTime` | `sync_agent` on a fresh agent (no prior import) behaves like `import_agent`; sets `created_premise: True` |
| `TestSyncIdempotency` | Double-syncing doesn't double-count removals; consecutive no-change syncs are stable |
| `TestSyncAgentRevocation` | After sync, the `test-agent:inactive` outlist guard still works — revoking `test-agent:active` cascades all beliefs OUT |
| `TestSyncOutWithJustifications` | JSON-path tests for OUT nodes that retain justifications: IN→OUT preserves them, OUT→IN clears retraction, re-sync of OUT-with-justifications is idempotent |
| `TestSyncRegistersRepo` | Sync records the agent's file directory in the repo registry (`api.list_repos`) |

## Patterns

**One-class-per-scenario.** Each class isolates a single sync behavior. Tests within a class are variations (e.g., IN→OUT and OUT→IN within `TestSyncTruthValueChanges`).

**Fixture composition.** `initial_import` depends on `db`, so tests that need a pre-populated DB use `initial_import`, while JSON-format tests use `db` directly and run their own import.

**Remote-as-string-mutation.** Tests mutate `INITIAL_BELIEFS` via `str.replace()` or string concatenation to produce the "updated remote" document. This makes diffs between initial and updated state visually obvious.

**Return-value-then-side-effect assertions.** Tests first check the summary dict (`result["beliefs_added"]`), then verify the actual node state via `api.show_node`. This validates both the reporting and the underlying mutation.

**Namespace-prefixed node IDs.** All `show_node` calls use `agent:id` form (e.g., `"test-agent:alpha-fact"`), confirming that sync namespaces beliefs under the agent name.

## Dependencies

**Imports:**
- `json` — for JSON-format test data
- `pytest` — fixtures and test infrastructure
- `reasons_lib.api` — the entire SUT; specifically `init_db`, `import_agent`, `sync_agent`, `show_node`, `retract_node`, `list_repos`
- `pathlib.Path` — used in one assertion (`TestSyncRegistersRepo`)

**Imported by:** Nothing — this is a leaf test module.

## Flow

A typical test follows this sequence:

1. `db` fixture creates an empty SQLite DB at a temp path.
2. `initial_import` writes `INITIAL_BELIEFS` to `beliefs.md` and calls `api.import_agent`, populating the DB with three namespaced nodes plus the agent's `active`/`inactive` control nodes.
3. The test modifies the markdown string (add/remove/change beliefs) and writes it to the same temp file.
4. `api.sync_agent("test-agent", path, db_path=db)` is called — this diffs the remote file against the DB and applies remote-wins reconciliation.
5. The test asserts on the returned summary dict (counts) and on individual node states via `api.show_node`.

For JSON tests, step 2 uses a `dict` serialized to `network.json` instead of markdown, and the import/sync calls are done inline rather than via the `initial_import` fixture.

## Invariants

1. **Remote wins.** If the remote file says IN and local is OUT (even from an explicit retraction), sync restores to IN. This is tested directly in `TestSyncRetractedCleared` and `TestSyncJson.test_sync_json_clears_retraction`.

2. **Cascade preservation.** After sync, the agent's outlist-based justification structure must remain intact — revoking `agent:active` must still cascade all agent beliefs to OUT. Tested in `TestSyncAgentRevocation`.

3. **Idempotency.** Syncing the same remote data twice produces zero changes on the second run. Tested in `TestSyncIdempotency` and `TestSyncOutWithJustifications.test_sync_out_with_justifications_idempotent`.

4. **STALE maps to OUT.** Beliefs marked `[STALE]` in the remote file are imported/synced as OUT nodes. Visible in `TestSyncNoChanges` where `beliefs_retracted == 1` for `gamma-stale`.

5. **First sync equals import.** `sync_agent` on an agent with no prior import produces the same result as `import_agent`, including `created_premise: True`. Tested in `TestSyncFirstTime`.

## Error Handling

There is no explicit error-handling testing in this file. The tests assume `sync_agent` succeeds and verify its return value and side effects. Missing nodes are not tested (e.g., what happens if `show_node` is called on a nonexistent ID). The implicit contract is that `sync_agent` raises on malformed input rather than returning partial results — but this file doesn't verify that boundary.

## Topics to Explore

- [function] `reasons_lib/api.py:sync_agent` — The implementation being tested; understanding the diff algorithm and reconciliation logic is essential
- [function] `reasons_lib/api.py:import_agent` — The initial-import path that `sync_agent` falls back to; compare their contracts
- [file] `tests/test_import_agent.py` — Companion tests for the import path; overlap and divergence with sync tests reveals which invariants are shared
- [general] `agent-namespace-and-control-nodes` — How `agent:active` and `agent:inactive` nodes gate all agent beliefs via outlist justifications
- [function] `reasons_lib/api.py:show_node` — The node inspection API used in every assertion; understanding its return shape (truth_value, justifications, metadata, text) is necessary to read these tests

## Beliefs

- `sync-agent-remote-wins` — `sync_agent` uses remote-wins semantics: a remote IN always overrides a local OUT, including explicit local retractions
- `sync-agent-idempotent` — Calling `sync_agent` twice with the same remote data produces zero adds, removes, and updates on the second call
- `sync-agent-preserves-cascade-structure` — After sync, agent revocation via `retract_node("agent:active")` still cascades all agent beliefs to OUT, proving justification outlist integrity is maintained
- `sync-agent-handles-both-formats` — `sync_agent` accepts both markdown (`beliefs.md`) and JSON (`network.json`) input formats with equivalent semantics
- `sync-agent-first-sync-equals-import` — When no prior import exists for an agent, `sync_agent` behaves identically to `import_agent`, returning `created_premise: True`

