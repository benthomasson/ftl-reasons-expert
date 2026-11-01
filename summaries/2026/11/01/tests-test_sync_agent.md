---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_sync_agent.py

# `tests/test_sync_agent.py` — Explanation

## 1. Purpose

This file is the **integration test suite for the `sync_agent` API**, which implements a "remote wins" synchronization strategy for agent beliefs. The core responsibility it owns is verifying that when an external beliefs file (markdown or JSON) changes after the initial `import_agent` call, the local database correctly:

- Adds new beliefs that appear in the remote
- Retracts beliefs that were removed from the remote
- Updates text, truth values, and dependencies of existing beliefs
- **Overrides local retractions** (the "remote wins" invariant)
- Preserves the agent-revocation cascade mechanism
- Behaves idempotently (re-syncing produces no side effects)

It does **not** test the underlying belief-network engine (that's `test_network.py`, `test_derive.py`, etc.) — it tests the *synchronization layer* that sits on top of the import/update primitives.

## 2. Key Components

### Constants

| Name | Role |
|------|------|
| `INITIAL_BELIEFS` | A 3-belief markdown document used as the canonical "remote state." Contains `alpha-fact` (IN, OBSERVATION), `beta-depends-alpha` (IN, DERIVED, depends on alpha), and `gamma-stale` (STALE/OBSERVATION, i.e., OUT). |

### Fixtures

- **`db`** — Creates a fresh SQLite DB at `tmp_path/"reasons.db"` via `api.init_db`. Each test gets an isolated database.
- **`initial_import`** — Writes `INITIAL_BELIEFS` to a temp `.md` file, calls `api.import_agent("test-agent", …)`, and returns `(db_path, tmp_path)`. All markdown-based tests consume this.

### Test Classes (grouped by behavior)

| Class | What it verifies |
|-------|-----------------|
| `TestSyncNoChanges` | Re-syncing identical content is a no-op; `gamma-stale` stays OUT. |
| `TestSyncNewBeliefs` | A new belief in the remote is added locally with correct metadata. |
| `TestSyncRemovedBeliefs` | A belief removed from the remote is retracted (OUT) locally. |
| `TestSyncUpdatedText` | Text changes propagate; `beliefs_updated` counter is accurate. |
| `TestSyncTruthValueChanges` | IN→OUT and STALE→IN transitions work; dependents cascade. |
| `TestSyncRetractedCleared` | **The core "remote wins" test**: local `retract_node` is overridden by sync. |
| `TestSyncUpdatedDependencies` | Changing `Depends on:` rewires justifications without losing truth. |
| `TestSyncJson` | Same behaviors but via JSON network format (add, update, remove, clear-retraction). |
| `TestSyncCountingAccuracy` | The result-dict counters (`beliefs_added/removed/updated/unchanged`) are exact. |
| `TestSyncFirstTime` | `sync_agent` on a brand-new agent acts like `import_agent` (sets `created_premise=True`). |
| `TestSyncIdempotency` | Two consecutive syncs with identical input produce zero mutations on the second. |
| `TestSyncAgentRevocation` | **Regression guard**: sync's internal `assert_node` must not strip `test-agent:inactive` from the outlist, or agent revocation would fail to cascade. |
| `TestSyncOutWithJustifications` | JSON path: OUT nodes *with* justifications are preserved (not deleted), and the transition is idempotent. |
| `TestSyncRegistersRepo` | After sync, `api.list_repos` shows the agent mapped to the file's parent directory. |

### The `sync_agent` Result Contract

Every test asserts against a dict returned by `api.sync_agent` with these keys:
- `beliefs_added` (int)
- `beliefs_removed` (int)
- `beliefs_retracted` (int)
- `beliefs_updated` (int)
- `beliefs_unchanged` (int)
- `created_premise` (bool, only on first-time sync)

## 3. Patterns

- **Arrange-Act-Assert**: Every test writes a modified file (Arrange), calls `api.sync_agent` (Act), then inspects both the result dict and `api.show_node` (Assert).
- **Fixture layering**: `db` is the base; `initial_import` builds on it. JSON tests skip `initial_import` and manage their own lifecycle because the JSON schema is structurally different.
- **Parametric state via file rewrites**: Instead of mocking, tests mutate the on-disk `.md` or `.json` file between import and sync — mirroring how the real system would receive an updated file.
- **Regression-pair tests**: `TestSyncAgentRevocation` and `TestSyncOutWithJustifications` exist specifically to prevent a class of bugs where sync's internal node-assertion logic corrupts the justification graph.
- **Format duality**: The same behavioral scenarios (add, remove, update, clear-retraction) are tested in both markdown and JSON to ensure the two import paths converge on the same sync logic.

## 4. Dependencies

**Imports (this file → others):**
- `json` — serializing JSON network fixtures
- `pytest` — fixture mechanism
- `reasons.api` — the SUT: `init_db`, `import_agent`, `sync_agent`, `show_node`, `retract_node`, `list_repos`
- `pathlib.Path` — resolving file paths for the repo-registration assertion

**Imported by:** Nothing (it's a leaf test module). It's discovered by pytest via the `tests/` directory convention.

**Implicit dependency:** `tests/conftest.py` may provide shared fixtures (e.g., LLM stubs), though this file's tests appear to be fully self-contained.

## 5. Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  Fixture setup                                                  │
│  1. tmp_path → reasons.db (api.init_db)                         │
│  2. Write INITIAL_BELIEFS to beliefs.md                          │
│  3. api.import_agent("test-agent", beliefs.md, db)              │
│     → creates agent:active, agent:inactive premises             │
│     → creates alpha-fact, beta-depends-alpha, gamma-stale nodes  │
│     → wires justifications, sets truth values                   │
└──────────────────────────────┬──────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  Test body (e.g., TestSyncRemovedBeliefs)                       │
│  1. Rewrite beliefs.md without beta-depends-alpha               │
│  2. api.sync_agent("test-agent", beliefs.md, db)               │
│     Internal: diff remote nodes vs. local store                 │
│       • beta-depends-alpha absent → retract (set OUT)           │
│       • alpha-fact unchanged → skip                             │
│       • gamma-stale unchanged → skip                            │
│     Returns {beliefs_removed: 1, ...}                           │
│  3. assert result["beliefs_removed"] == 1                       │
│  4. api.show_node("test-agent:beta-depends-alpha")             │
│     → truth_value == "OUT"                                      │
└─────────────────────────────────────────────────────────────────┘
```

The critical internal step (not visible here but implied) is that `sync_agent` calls `assert_node` for each remote belief. That call must preserve the `outlist` entries referencing `test-agent:inactive` — if it doesn't, the agent-revocation cascade breaks (the regression the last two test classes guard against).

## 6. Invariants

| Invariant | Where enforced |
|-----------|---------------|
| **Remote wins**: A locally-retracted belief is restored to its remote truth value on the next sync. | `TestSyncRetractedCleared`, `TestSyncJson.test_sync_json_clears_retraction` |
| **Cascade**: Changing a premise's truth value propagates to all derived dependents. | `TestSyncTruthValueChanges.test_sync_in_to_out` (beta follows alpha) |
| **Idempotency**: Syncing the same state twice produces zero mutations on the second call. | `TestSyncIdempotency`, `TestSyncOutWithJustifications.test_sync_out_with_justifications_idempotent` |
| **Agent-revocation integrity**: Every justification's outlist must contain `test-agent:inactive` after sync. | `TestSyncAgentRevocation` |
| **First-sync equivalence**: `sync_agent` on a fresh agent ≡ `import_agent` (sets `created_premise=True`). | `TestSyncFirstTime` |
| **Counter accuracy**: `beliefs_updated == 0` when nothing changed; justification-only changes count as updates. | `TestSyncCountingAccuracy` |
| **Repo registration**: After any sync, `list_repos` maps the agent to the file's directory. | `TestSyncRegistersRepo` |

## 7. Error Handling

This test file **does not test error paths** (no invalid markdown, no missing files, no concurrent writes). All tests assume well-formed input and assert on success-path return values. Error handling in `sync_agent` (e.g., what happens if the file is unparseable, or if a dependency references a non-existent node) would be tested elsewhere or is a gap.

The one implicit error-handling contract visible here: `sync_agent` **returns** a result dict rather than raising. The tests never use `pytest.raises`, confirming the function signals outcomes through the dict's counters and node state rather than exceptions.

---

## Topics to Explore

- [function] `reasons/api.py:sync_agent` — The SUT under test; understanding its diff-and-apply algorithm (how it computes added/removed/updated sets and calls `assert_node`) is essential for maintaining these tests.
- [function] `reasons/api.py:assert_node` — The primitive that sync calls per-belief; the agent-revocation regression tests exist because of a subtle invariant in this function's outlist handling.
- [file] `reasons/import_agent.py` — The initial-import path; `sync_agent` likely reuses or parallels its parsing logic, so understanding both reveals where format-specific (md vs. JSON) branching lives.
- [general] `belief-network-truth-propagation` — How OUT cascades through `Derivable` justifications; the tests assert this but the engine lives in `reasons/network.py`.
- [file] `tests/conftest.py` — Shared fixtures and any LLM-mocking setup that might affect `sync_agent` if it ever calls the LLM for summarization or conflict resolution.

---

## Beliefs

- `sync-agent-implements-remote-wins` — `sync_agent` unconditionally overwrites local truth values with the remote file's stated values; a local `retract_node` call is erased on the next sync.
- `sync-agent-preserves-agent-revocation-cascade` — Every justification's outlist must retain the `test-agent:inactive` entry after a sync, or revoking the agent will fail to cascade OUT to its beliefs.
- `sync-agent-is-idempotent` — Calling `sync_agent` twice with identical remote content produces zero mutations (all counters are 0) on the second invocation.
- `sync-agent-equivalent-to-import-on-first-call` — When no prior import exists, `sync_agent` sets `created_premise=True` and behaves identically to `import_agent`.
- `sync-agent-returns-counters-not-exceptions` — `sync_agent` communicates outcomes exclusively through a result dict (`beliefs_added`, `beliefs_removed`, `beliefs_updated`, `beliefs_unchanged`, `beliefs_retracted`); it does not raise on expected-state transitions.
