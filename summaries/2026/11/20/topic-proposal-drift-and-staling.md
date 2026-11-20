---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: The optimistic-concurrency model this suite pins: `(action, target)` staling + drift-to-stale on accept, and how callers are expected to react.

## Optimistic-concurrency model

The suite pins a simple optimistic-write model for proposals:

1. **Proposal creation does not lock the target.**
2. **Proposals are invalidated by `(action, target)` conflict.**
3. **Accept performs a compare-and-swap-like re-validation against the live network.**
4. **If the live state has drifted from the proposal snapshot, the proposal becomes `stale` and is not applied.**
5. **Callers must treat `stale` as a normal concurrency outcome and re-read/re-propose, not blindly retry the same proposal.**

### 1. Proposal-time staling is keyed by `(action, target)`

`_stale_older_pending(conn, target_id, action, now)` in `reasons/api.py:5199` is the clearest expression of the model:

- Its docstring says: “Mark older pending proposals for the same `(target, action)` as stale.” (`reasons/api.py:5200`)
- It executes an update setting `status = 'stale'` for older pending proposals (`reasons/api.py:5209`)
- It returns the list of proposal IDs it staled (`reasons/api.py:5213`)

So pending proposals are not treated as a durable queue for the same action-target pair. A newer proposal supersedes older pending proposals for that pair. That is the suite’s “last valid proposal wins” rule: old proposals remain in the table as `stale`, but callers should no longer treat them as acceptable work.

The higher-level summary at `reasons/api.py:5232` says older pending proposals for the same target are auto-staled, but the helper signature/docstring at `5199-5200` makes the real key explicit: **`(target_id, action)`**, not merely target.

### 2. Accept-time drift is a snapshot-vs-live comparison

The accept path is described around `reasons/api.py:5631-5633`:

- “Re-validates against the live network to detect drift.”
- If successful, the proposal is accepted.
- “On drift, marks it stale without applying.”

The proposal therefore carries a snapshot of the target state from proposal time. At accept time the code compares that snapshot to the current network. The observed truth-value drift messages at `reasons/api.py:5719` and `reasons/api.py:5740` explicitly compare `snapshot['truth_value']` against the current value:

```text
Truth value drifted: was {snapshot['truth_value']}, ...
```

The tests pin the concrete drift cases:

- `tests/test_api.py:1240` — `test_drift_target_already_out`
- `tests/test_api.py:1249` — `test_drift_target_deleted`
- `tests/test_api.py:1256` — `test_drift_add_node_already_exists`
- `tests/test_api.py:1263` — `test_drift_supersede_new_id_collision`

These cover the expected optimistic-concurrency failures:

| Drift case | Expected outcome |
|---|---|
| Target already OUT when proposal expected otherwise | stale, not applied |
| Target deleted since proposal | stale, not applied |
| Add-node target already exists | stale, not applied |
| Supersede new ID already collides | stale, not applied |
| Target truth value changed | stale, not applied |

The same contract is exposed through other layers:

- CLI: `tests/test_cli.py:1454` — `test_accept_stale_drift`
- MCP: `tests/test_mcp_server.py:555` — `test_accept_drift_already_out`
- MCP documentation at `reasons/mcp_server.py:485-488`: “Accept a pending proposal — applies the change after drift re-validation… if [the target] has changed since the proposal was created (drift), the proposal is [stale].”

### 3. Caller expectations

The model implies the following caller contract:

- **On proposal creation:** if older pending proposals are returned as staled, stop treating them as live. They were superseded by a newer proposal for the same `(action, target)`.
- **On accept success:** the change was applied.
- **On accept `stale`:** do **not** retry the same proposal ID blindly. The precondition no longer holds, the target was deleted, the target already reached the desired state, or a newer proposal superseded it.
- **After `stale`:** re-read current network state, then decide whether to:
  - re-propose from the new state,
  - treat the desired state as already achieved, e.g. retract target already OUT or add node already exists,
  - or abandon the change.
- **At API/CLI/MCP boundaries:** `stale` should be surfaced as a normal result, not converted into an opaque failure. The tests at `tests/test_cli.py:1454` and `tests/test_mcp_server.py:555` exist precisely because callers must be able to observe the stale drift outcome.

This is the key difference from pessimistic locking: between propose and accept, the system does not hold a lock. It assumes conflicts are rare and resolves them at accept time by marking the losing/drifted proposal `stale`.

### 4. Important disambiguation

There is another “stale” concept in the codebase: source-file staleness in `reasons/check_stale.py`. That module hashes source files and compares stored `source_hash` against current SHA-256 hashes (`reasons/check_stale.py:1-14`, `check_stale` around line 57). That is about whether a belief’s source material changed. The optimistic-concurrency model described here is about **proposal status** in the `proposals` table: `(action, target)` staling and accept-time drift-to-stale.

### 5. What the observations do not fully show

The exact accept response schema is not visible in these observations. We know the behavior is “mark stale without applying” (`reasons/api.py:5631-5633`) and that callers observe stale drift through CLI/MCP tests, but we do not see the precise returned fields, status strings, or stale-reason payload. To pin that down, inspect the full accept implementation around `reasons/api.py:5631-5740` and the assertions in `tests/test_api.py:1240-1263`.

## Topics to Explore

- [function] `reasons/api.py:_stale_older_pending` — Shows the exact SQL and return semantics for proposal-time `(target, action)` staling.
- [file] `reasons/api.py` — Inspect the full accept path around lines 5631–5740 for drift detection and stale marking.
- [file] `tests/test_api.py` — Pins the concrete drift cases: target deleted, already OUT, add-node existing, supersede ID collision.
- [file] `reasons/mcp_server.py` — Documents how the MCP accept tool exposes drift re-validation and stale outcomes.
- [general] `proposal-status-lifecycle` — Explore pending/stale/accepted proposal states and how CLI, API, and MCP callers react to each.

## Beliefs

- `proposal-staling-is-keyed-by-action-and-target` — `_stale_older_pending` in `reasons/api.py:5199` takes `target_id` and `action` and marks older pending proposals for that same pair as `stale`.
- `accept-revalidates-drift-before-apply` — `reasons/api.py:5631-5633` re-validates against the live network and marks a proposal stale without applying it when drift is detected.
- `truth-value-drift-compares-snapshot-to-current` — `reasons/api.py:5719` and `:5740` emit a “Truth value drifted” message comparing `snapshot['truth_value']` to the current value.
- `drift-cases-are-pinned-by-api-tests` — `tests/test_api.py:1240-1263` covers target already OUT, target deleted, add-node already exists, and supersede ID collision as drift-to-stale cases.
- `mcp-accept-doc-requires-drift-revalidation` — `reasons/mcp_server.py:485-488` documents accept as applying a change only after drift re-validation and treating drifted proposals as stale.
