# File: tests/test_sync_agent.py

**Date:** 2026-04-24
**Time:** 16:53

# `tests/test_sync_agent.py` — Sync Agent Test Suite

## 1. Purpose

This file tests `api.sync_agent()`, the operation that re-imports an agent's belief file and reconciles it against what's already in the database. The core semantic is **remote wins**: when the external file (markdown or JSON) disagrees with the local database state, the file is authoritative. This is the incremental counterpart to `api.import_agent()` — import is the initial load, sync is every subsequent update.

The file owns verification of the complete sync contract: additions, removals, text updates, truth-value changes, dependency rewiring, idempotency, counting accuracy, and — critically — that sync doesn't break the TMS invariant that agent revocation cascades correctly.

## 2. Key Components

### `INITIAL_BELIEFS` (module-level constant)

A beliefs.md fixture containing three nodes that exercise distinct archetypes:
- **`alpha-fact`** — an IN premise (OBSERVATION), no dependencies
- **`beta-depends-alpha`** — an IN derived node, depends on `alpha-fact`
- **`gamma-stale`** — a STALE observation, which the import system maps to OUT

This trio is carefully chosen: it gives you a dependency chain (alpha → beta) and a retracted node (gamma) so that every test can probe cascading behavior without extra setup.

### Fixtures

- **`db`** — creates a fresh SQLite database via `api.init_db()` in a temp directory. Every test gets an isolated DB.
- **`initial_import`** — writes `INITIAL_BELIEFS` to disk and calls `api.import_agent("test-agent", ...)`. Returns `(db_path, tmp_path)` so tests start from a known imported state.

### Test Classes (organized by sync scenario)

| Class | What it tests |
|---|---|
| `TestSyncNoChanges` | Syncing identical content is a no-op (except gamma stays OUT) |
| `TestSyncNewBeliefs` | New nodes in the file get added to the DB |
| `TestSyncRemovedBeliefs` | Nodes absent from the file get retracted (OUT) |
| `TestSyncUpdatedText` | Changed belief text propagates to the DB |
| `TestSyncTruthValueChanges` | IN→OUT and OUT→IN transitions, including cascade to dependents |
| `TestSyncRetractedCleared` | Local retraction overridden by remote IN ("remote wins") |
| `TestSyncUpdatedDependencies` | Rewiring a node's `Depends on:` updates its justification antecedents |
| `TestSyncJson` | Same sync scenarios but with JSON input format instead of markdown |
| `TestSyncCountingAccuracy` | The returned dict counts (`beliefs_added`, `beliefs_updated`, etc.) are correct |
| `TestSyncFirstTime` | Calling `sync_agent` when no prior import exists works like `import_agent` |
| `TestSyncIdempotency` | Two consecutive syncs produce stable results; removals aren't double-counted |
| `TestSyncAgentRevocation` | **Regression guard**: after sync, retracting `agent:active` still cascades all beliefs OUT |

## 3. Patterns

**Remote-wins reconciliation**: Every test manipulates the file, then calls `sync_agent`, then inspects DB state. The file is always treated as the source of truth.

**Fixture layering**: `initial_import` depends on `db`, giving two entry points — tests that need a pre-populated DB use `initial_import`; tests that need a bare DB (like `TestSyncJson`, `TestSyncFirstTime`) use `db` directly.

**One class per scenario**: Each class isolates a single dimension of sync behavior. This makes failures immediately diagnostic — a failing class tells you exactly which sync capability broke.

**Namespaced node IDs**: All nodes are stored as `agent-name:node-id` (e.g., `test-agent:alpha-fact`). Tests reference this composite key when calling `api.show_node()`.

**Result-dict assertions**: `sync_agent` returns a dict with keys like `beliefs_added`, `beliefs_removed`, `beliefs_updated`, `beliefs_unchanged`. Tests assert on these counts as a contract — callers depend on accurate accounting.

## 4. Dependencies

**Imports**:
- `json` — used only by `TestSyncJson` to build JSON fixtures
- `pytest` — fixtures and test infrastructure
- `reasons_lib.api` — the entire public API surface under test: `init_db`, `import_agent`, `sync_agent`, `show_node`, `retract_node`

**Imported by**: Nothing. This is a leaf test module.

**Implicit dependencies**: `sync_agent` internally uses `import_agent`'s parsing (markdown via `import_beliefs`, JSON via `import_json`), the TMS propagation engine in `network.py`, and SQLite persistence in `storage.py`. Failures here can originate from any of those layers.

## 5. Flow

A typical test follows this pattern:

1. **Setup**: `db` fixture creates a fresh SQLite DB. `initial_import` writes `INITIAL_BELIEFS` to a file and calls `api.import_agent()`, populating the DB with three nodes.
2. **Mutate**: The test modifies the beliefs file — adding, removing, or changing nodes.
3. **Sync**: Calls `api.sync_agent("test-agent", file_path, db_path=db)`.
4. **Assert on result dict**: Checks counts like `beliefs_added`, `beliefs_removed`.
5. **Assert on DB state**: Calls `api.show_node()` to verify truth values, text, justifications, and metadata match expectations.

For cascade tests (`test_sync_in_to_out`, `TestSyncAgentRevocation`), step 5 also checks dependent nodes to verify propagation integrity.

## 6. Invariants

- **Remote wins**: If the file says IN and the DB says OUT, the DB must change to IN after sync. This is tested explicitly in `TestSyncRetractedCleared` and `TestSyncTruthValueChanges.test_sync_out_to_in`.
- **Cascade preservation**: Syncing must not detach nodes from their justification outlists. `TestSyncAgentRevocation` is a regression test for this — it verifies that after sync, retracting `agent:active` still cascades all agent beliefs to OUT.
- **Idempotency**: Syncing the same file twice produces `beliefs_added == 0, beliefs_removed == 0` on the second call.
- **STALE maps to OUT**: Nodes marked `[STALE]` in the beliefs file are imported/synced with truth value OUT.
- **Agent namespace isolation**: All node IDs are prefixed with the agent name, so `sync_agent("test-agent", ...)` only touches `test-agent:*` nodes.
- **First sync = import**: `sync_agent` on a previously unknown agent name must behave identically to `import_agent`.

## 7. Error Handling

This file doesn't test error paths — it focuses on the happy path of the sync contract. There are no tests for malformed markdown, missing files, or conflicting agent names. Errors from the underlying API would surface as unhandled exceptions, which pytest would report as test failures rather than graceful error handling.

---

## Topics to Explore

- [function] `reasons_lib/api.py:sync_agent` — The implementation being tested; understanding the diff-and-reconcile algorithm is essential to maintaining these tests
- [function] `reasons_lib/api.py:import_agent` — The initial import path that `sync_agent` builds on; shares parsing logic but has different semantics for existing nodes
- [file] `reasons_lib/import_beliefs.py` — The markdown parser that converts `beliefs.md` format into structured node data; changes here directly affect what sync sees
- [function] `reasons_lib/network.py:propagate` — The BFS propagation engine that makes cascading retraction/restoration work after sync modifies truth values
- [general] `agent-active-outlist-pattern` — How the `agent:active` / `agent:inactive` sentinel nodes control bulk revocation of an agent's beliefs via outlist membership

## Beliefs

- `sync-agent-remote-wins` — `sync_agent` always treats the external file as authoritative: if the file says IN and the DB says OUT, the node becomes IN
- `sync-agent-first-call-equals-import` — Calling `sync_agent` for an agent with no prior import produces identical results to `import_agent`, including setting `created_premise: True`
- `sync-agent-idempotent` — Two consecutive `sync_agent` calls with the same file content produce zero additions, removals, and updates on the second call
- `sync-preserves-cascade-wiring` — After `sync_agent` runs, the `agent:inactive` outlist entries on beliefs are preserved, so retracting `agent:active` still cascades all beliefs to OUT
- `stale-maps-to-out-on-import` — Nodes marked `[STALE]` in beliefs.md are stored with truth value OUT; this holds for both initial import and subsequent syncs

