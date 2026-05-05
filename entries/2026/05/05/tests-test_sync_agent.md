# File: tests/test_sync_agent.py

**Date:** 2026-05-05
**Time:** 15:27

## Purpose

`tests/test_sync_agent.py` is the test suite for the `api.sync_agent()` function — the **remote-wins synchronization** mechanism for multi-agent belief federation. It verifies that when an external agent's belief file (markdown or JSON) is re-imported, the local TMS database correctly reconciles differences: adding new beliefs, retracting removed ones, updating changed text/dependencies, and respecting truth-value changes from the remote source.

This file owns the contract that **sync is a correct, idempotent, remote-authoritative merge operation** on the belief network.

## Key Components

### Test Data

**`INITIAL_BELIEFS`** (lines 10–25) — A markdown string defining a three-belief baseline:
- `alpha-fact` — an IN premise (observation)
- `beta-depends-alpha` — an IN derived belief that depends on `alpha-fact`
- `gamma-stale` — a STALE observation (imported as OUT)

This small network is carefully designed: it includes a dependency chain (beta → alpha), a non-monotonic state (STALE → OUT), and enough variety to exercise all sync paths.

### Fixtures

- **`db`** — Creates a fresh SQLite database in `tmp_path` via `api.init_db()`. Every test gets an isolated store.
- **`initial_import`** — Writes `INITIAL_BELIEFS` to disk and runs `api.import_agent("test-agent", ...)`, returning `(db_path, tmp_path)`. This is the "before" state that most sync tests diff against.

### Test Classes (10 classes, ~20 tests)

| Class | What it tests |
|---|---|
| `TestSyncNoChanges` | No-op sync returns zero adds/removes, gamma still OUT |
| `TestSyncNewBeliefs` | New belief in remote → added locally with correct metadata |
| `TestSyncRemovedBeliefs` | Belief missing from remote → retracted locally (OUT) |
| `TestSyncUpdatedText` | Changed description text → updated in local node |
| `TestSyncTruthValueChanges` | IN→OUT cascades to dependents; STALE→IN restores |
| `TestSyncRetractedCleared` | Local retraction overridden by remote IN ("remote wins") |
| `TestSyncUpdatedDependencies` | Changed `Depends on:` rewires justification antecedents |
| `TestSyncJson` | Same sync semantics over JSON (`network.json`) format |
| `TestSyncCountingAccuracy` | Result dict counts (`beliefs_added`, `beliefs_updated`, `beliefs_unchanged`) are accurate |
| `TestSyncFirstTime` | Sync on a never-imported agent works as initial import |
| `TestSyncIdempotency` | Double-sync is stable: second run is a no-op |
| `TestSyncAgentRevocation` | After sync, retracting `agent:active` still cascades OUT to all agent beliefs |
| `TestSyncRegistersRepo` | Sync records the agent's file path in the repo registry |

## Patterns

**Fixture composition** — `initial_import` depends on `db`, so tests that need the baseline use `initial_import`; tests that need a bare database (JSON tests, first-time sync) use `db` directly.

**Mutate-and-sync** — Every test follows the same pattern:
1. Start from known state (fixture)
2. Write a modified belief file to `tmp_path`
3. Call `api.sync_agent()` 
4. Assert on the return dict (counts) and/or on individual nodes via `api.show_node()`

**Remote-wins semantics** — The file is structured around the principle that the remote file is authoritative. `TestSyncRetractedCleared` and `TestSyncJson.test_sync_json_clears_retraction` are the key proof: a locally-retracted belief is restored if the remote still says IN.

**Agent-namespaced node IDs** — All `show_node` calls use `"agent:belief-id"` format (e.g., `"test-agent:alpha-fact"`), confirming that the import/sync system namespaces beliefs by agent.

**Cascade verification** — Several tests check that TMS propagation still works after sync. When `alpha-fact` goes OUT, `beta-depends-alpha` must also go OUT. This validates that sync doesn't break justification wiring.

## Dependencies

**Imports:**
- `json` — for JSON format sync tests
- `pytest` — test framework and fixtures
- `reasons_lib.api` — the entire public API surface under test: `init_db`, `import_agent`, `sync_agent`, `show_node`, `retract_node`, `list_repos`
- `pathlib.Path` — used once in `TestSyncRegistersRepo` for path resolution

**Imported by:** Nothing — this is a leaf test module.

## Flow

A typical test execution:

1. `db` fixture → `api.init_db()` creates empty SQLite at `tmp_path/reasons.db`
2. `initial_import` fixture → writes markdown to file, calls `api.import_agent()` which parses beliefs, creates namespaced nodes with justifications (including `agent:inactive` in outlists for GATE-style agent revocation)
3. Test body → modifies the markdown/JSON file on disk
4. `api.sync_agent()` → diffs remote file against local DB state, applies changes (add/retract/update/rewire), returns a summary dict
5. Assertions check both the summary dict and the actual node states in the DB

## Invariants

1. **Sync is idempotent** — `TestSyncIdempotency` proves two consecutive syncs with identical data produce the same result, with the second being a no-op.
2. **Remote always wins** — If the remote says IN and local says OUT (even via explicit retraction), the node is restored to IN.
3. **Cascade integrity survives sync** — After sync, retracting `agent:active` must cascade OUT to all beliefs. Sync must not detach justification wiring (the `agent:inactive` outlist entry).
4. **Counting is accurate** — The return dict must not double-count: a belief removed in sync 1 must show `beliefs_removed == 0` in sync 2.
5. **Agent namespace isolation** — Nodes are always accessed as `"agent:belief-id"`, never bare.
6. **STALE imports as OUT** — A `[STALE]` belief in the remote file enters the local DB with truth value OUT.

## Error Handling

This file tests the happy path exclusively — no tests for malformed markdown, missing files, or concurrent access. Error handling is implicit through pytest: if `api.sync_agent()` raises, the test fails. The fixture design (isolated `tmp_path` + fresh DB) prevents cross-test contamination.

## Topics to Explore

- [function] `reasons_lib/api.py:sync_agent` — The implementation being tested; understanding the diff/merge algorithm is essential to understanding why these tests are structured this way
- [function] `reasons_lib/api.py:import_agent` — The initial import path that `sync_agent` builds on; explains how agent namespacing and the `agent:inactive` outlist pattern work
- [file] `tests/test_import_agent.py` — Companion tests for the initial import path; sync tests assume import works correctly
- [general] `agent-revocation-pattern` — How the `agent:active` / `agent:inactive` outlist mechanism enables whole-agent retraction cascades
- [file] `reasons_lib/import_agent.py` — The markdown/JSON parser that both import and sync use to extract beliefs from files

## Beliefs

- `sync-agent-is-remote-wins` — `api.sync_agent()` implements remote-wins semantics: the remote file is authoritative over local state, including overriding local retractions
- `sync-agent-is-idempotent` — Calling `sync_agent` twice with the same data produces identical results; the second call is a no-op with zero adds/removes/updates
- `sync-preserves-cascade-wiring` — After sync, justification antecedents and outlist entries (including `agent:inactive`) remain intact, so agent revocation still cascades correctly
- `sync-supports-markdown-and-json` — `sync_agent` accepts both markdown (`beliefs.md`) and JSON (`network.json`) formats with the same remote-wins semantics
- `stale-beliefs-import-as-out` — Beliefs marked `[STALE]` in the remote file are imported/synced with truth value OUT, counted under `beliefs_retracted`

