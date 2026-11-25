---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Trace where `_retracted` is read and written across the codebase; it underpins the "would be IN if not for `_retracted`" premise of this migration.

## Trace of `_retracted`

### Storage: a metadata boolean, not a schema column

`_retracted` is **not** a dedicated column. It lives inside `node.metadata` and is persisted via `metadata_json` in the `nodes` table (`reasons/storage.py`, `SCHEMA` → `metadata_json TEXT DEFAULT '{}'`, saved at `Storage.save` and rehydrated at `Storage.load`). It is distinct from the `retracted_at TEXT` column, which stores a *timestamp* string, not the boolean latch. So `_retracted` survives save/load as ordinary JSON metadata.

### Writes

| Location | What happens |
|---|---|
| `reasons/network.py:163` | `node.metadata["_retracted"] = True` — the core primitive. Almost certainly inside `Network.retract()`, set alongside the truth-value flip to OUT. This is the canonical write. |
| `reasons/import_agent.py:311` | `network.nodes[node_id].metadata["_retracted"] = True` — during agent import, claims that map to STALE/OUT source status get stamped retracted (paired with the `claims_retracted` counter at line 330 and `import_beliefs.py:298`). |
| `reasons/import_agent.py:411-412` | Idempotent set: `if not node.metadata.get("_retracted"): ... = True` on the belief leg when source says OUT/STALE. |
| `reasons/import_agent.py:467` | Explicit set with a comment (lines 464-466) explaining it's set *directly instead of `retract_after`* "to keep OUT-with-justifications nodes locked." |
| `reasons/import_agent.py:477-478` | `elif not node.metadata.get("_retracted"): ... = True` in the belief-status loop; `499` increments `beliefs_retracted`. |

### Reads

| Location | What happens |
|---|---|
| `reasons/import_agent.py:411` | `if not node.metadata.get("_retracted")` — guard before setting, i.e. read-then-write. |
| `reasons/import_agent.py:425` | `if node.metadata.get("_retracted"): ...` then `pop` at 426 — clears the flag when the remote explicitly sends IN. The surrounding comment at 402-404 spells out the intent: "remote flips it to IN. `_retracted` is set so both `_propagate()`..." |
| `reasons/import_agent.py:477` | `elif not node.metadata.get("_retracted")` — same guard on the belief leg. |

The critical consumer of the flag is `_propagate()`, referenced only in prose (`import_agent.py:402-404`) — **that read site is not present in the observations** (see Gaps below).

### Clears (the migration's `pop`)

- `reasons/api.py:1637` — `node.metadata.pop("_retracted", None)`. This is the migration exit path: after `migrate_retract_to_defeaters` attaches a `migrated-retraction` defeater, it removes the flag so the node is no longer latched OUT by the boolean.
- `reasons/import_agent.py:426` and `:485` — clears driven by the remote source saying IN.

### How this underpins the "would be IN if not for `_retracted`" premise

The migration test (`tests/test_migrate_defeaters.py`) pins down the contract:

1. **`test_basic_migration`** — a `derived` node with a satisfied SL justification is IN; `retract_node` forces it OUT; migration attaches `migrated-retraction-derived-j0` and leaves truth_value OUT. The defeater *now* is what keeps the node OUT, so the underlying justification is once again load-bearing.
2. **`test_metadata_cleared`** — after migration, `assert "_retracted" not in node["metadata"]`. The flag must be gone, which is the `api.py:1637` pop.
3. **`test_reversibility`** — retracting the migrated defeater flips the belief back to IN. This is the operational proof of the premise: nothing structural about the belief prevented it from being IN, only the `_retracted` latch — and once that is replaced by a defeater, removing the defeater restores IN.

The docstring fragment at `reasons/api.py:1557` states it directly: candidates are nodes "satisfied if not for the `_retracted` flag." The migration therefore has to (a) read `_retracted` to identify candidates, (b) confirm the justifications *would* satisfy, (c) create the defeater, (d) `pop("_retracted")` at 1637. Steps (b) and the "would satisfy" check are the exact predicate the tests encode.

### Gaps in the observations

- **`Network._propagate()` is never shown.** The comments in `import_agent.py:402-404` and `467` assert that `_propagate()` reads `_retracted` to keep the node OUT despite satisfied justifications. Without seeing `network.py` around line 163 (the `retract` method) and the `_propagate` body, we cannot state *how* the read forces the truth value — only that the flag exists and is inverted at migration time.
- **`api.py:migrate_retract_to_defeaters` body is missing.** We see the docstring fragment at 1557 and the `pop` at 1637, but not the candidate-selection logic that must read `_retracted` (probably `node.metadata.get("_retracted")` or the inverse of satisfied-justification checks).
- **No hits for `"would be IN"`** (`find_migration_premise_text` returned 0) — the premise is phrased in code as "satisfied if not for the `_retracted` flag" (api.py:1557), not in the literal wording of this question.
- Full grep returned 101 matches but only ~27 are enumerated; the rest (e.g. other `metadata.get("_retracted")` reads in propagation) are truncated.

## Topics to Explore

- [file] `reasons/network.py` — Locate the `retract()` method around line 163 and the `_propagate()` body to see the read that forces OUT while `_retracted` is set.
- [file] `reasons/api.py` — Read `migrate_retract_to_defeaters` (candidate selection at ~1557, `pop` at 1637) to see how "would be IN" is computed across `justifications`.
- [function] `reasons/import_agent.py:_import_beliefs` — How `_retracted` is re-synced on incremental imports (lines 400-500).
- [file] `tests/test_migrate_defeaters.py` — The exact behavioural contract for the flag's removal and reversibility.
- [general] `retracted-vs-retracted_at` — Disambiguate the `metadata["_retracted"]` boolean from the `retracted_at` timestamp column in `storage.py`.

## Beliefs

- `retracted-is-stored-in-metadata` — `_retracted` is a boolean in `node.metadata`, persisted in `metadata_json`, not a dedicated `nodes` column (separate from `retracted_at TEXT`).
- `network-retract-sets-retracted-flag` — `reasons/network.py:163` is the canonical write that sets `metadata["_retracted"] = True` during retraction.
- `migration-clears-retracted-flag` — `migrate_retract_to_defeaters` removes the flag via `node.metadata.pop("_retracted", None)` at `reasons/api.py:1637`.
- `migration-preserves-out-through-defeater` — After migration the node stays OUT because the new `migrated-retraction-*` defeater holds it OUT, not the `_retracted` flag (per `test_basic_migration` + `test_metadata_cleared`).
- `retracting-migrated-defeater-restores-in` — Removing the migrated defeater flips the belief back to IN, proving the justification was satisfied all along (`test_reversibility`).
- `import-agent-reads-retracted-for-resync` — `reasons/import_agent.py` reads `_retracted` at lines 411, 425, and 477 to decide whether to stamp or clear the flag on incremental sync.
