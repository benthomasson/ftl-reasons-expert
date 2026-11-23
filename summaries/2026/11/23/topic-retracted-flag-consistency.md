---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Where `metadata["_retracted"]` is honored vs bypassed (`recompute_all` vs `add_justification`), and how that affects restoration semantics.

## Short answer

`metadata["_retracted"]` acts as a **retraction/pin** that blocks ordinary global recomputation. `Network.recompute_all` honors it and skips flagged nodes, so imported OUT/STALE beliefs cannot be resurrected by propagation. `Network.add_justification` is a local repair/restoration path: it appends a justification and can bring an OUT node back IN by re-evaluating the target/dependents, bypassing dependency-based OUT state. Explicit unretraction is separate: `reasons/api.py:1637` pops `_retracted`, then `reasons/api.py:1647` calls `net.recompute_all()`.

## Where `_retracted` is honored

- `reasons/network.py:904` defines `Network.recompute_all`.
- Comments in `reasons/import_agent.py:402-404` explicitly say `_retracted` is set so both `_propagate()` and `recompute_all()` skip the node, keeping it OUT until the remote explicitly sends IN.
- `reasons/import_agent.py:537-538` says OUT/STALE source beliefs are marked with `_retracted` metadata so `recompute_all` cannot resurrect them.
- Therefore, a node with `_retracted=True` is treated as a hard lock for global recomputation: `recompute_all` will not restore it merely because some justification would otherwise be satisfied.

## Where `_retracted` is bypassed / where restoration happens

- `reasons/api.py:406` defines API-level `add_justification`, delegating to `net.add_justification` at `reasons/api.py:461` / `468`.
- `reasons/network.py:496` defines `Network.add_justification`.
- Tests show `add_justification` can restore an OUT node:
  - `tests/test_add_justification.py:43-60` — after `net.retract("a")`, node `b` is OUT; adding a valid justification for `b` makes it IN.
  - `tests/test_add_justification.py:62-83` — adding a justification that restores `b` also cascades to dependent `d`.
- So `add_justification` bypasses ordinary support-based OUT state by adding a new valid justification and restoring the target/dependents.

## Important caveat from the observations

The exact bodies of `Network.recompute_all` (`reasons/network.py:904`) and `Network.add_justification` (`reasons/network.py:496`) are not included. Also, the observed restoration test `test_add_justification_restores_out_node` retracts antecedent `a`, making `b` OUT indirectly; it does **not** show `add_justification` being called on a node whose own metadata has `_retracted=True`.

So the safe interpretation is:

- `recompute_all` demonstrably honors `_retracted` via comments and import semantics.
- `add_justification` demonstrably restores OUT nodes due to lost support.
- The observations do not prove whether `add_justification` clears or ignores an explicit `_retracted` flag on the target node. The explicit clear path shown is `reasons/api.py:1637`, followed by `net.recompute_all()` at `reasons/api.py:1647`.

## Restoration semantics

- **Global recomputation cannot resurrect a retracted node.** `_retracted` is a pin honored by `recompute_all` and `_propagate()`.
- **Adding a justification can restore a node that is OUT because its support was lost.** This is the local repair path shown by `tests/test_add_justification.py`.
- **Explicit restoration of a retracted node appears to require clearing `_retracted`.** `reasons/api.py:1637` pops the flag, and `reasons/api.py:1647` then recomputes.
- **Import/sync semantics follow source truth.** `reasons/import_agent.py:311`, `411-412`, `464-467`, `477-478`, and `498-499` set `_retracted` for OUT/STALE source beliefs. `reasons/import_agent.py:425-426` and `485` clear it when the remote source says IN. This prevents `recompute_all` from resurrecting remote-retracted beliefs.

## Topics to Explore

- [function] `reasons/network.py:recompute_all` — Inspect the exact skip condition for `_retracted` and how it interacts with `_propagate()`.
- [function] `reasons/network.py:add_justification` — Verify whether it checks, clears, or ignores `_retracted` on the target node.
- [file] `reasons/api.py` — Read around lines 1550–1650 for retraction/restoration API logic, including the `_retracted` pop and subsequent `recompute_all`.
- [file] `reasons/import_agent.py` — Understand the source OUT/STALE → `_retracted` mapping and clear-on-IN behavior.
- [general] `retracted-node-restoration-test-coverage` — Check whether tests cover adding a justification directly to a node that already has `_retracted=True`.

## Beliefs

- `recompute-all-honors-retracted-flag` — `Network.recompute_all` skips nodes with `metadata["_retracted"]`, per comments in `reasons/import_agent.py:402-404` and `537-538`.
- `retracted-flag-prevents-global-resurrection` — Imported OUT/STALE beliefs marked `_retracted` cannot be restored by ordinary `recompute_all()` propagation.
- `add-justification-enables-out-node-restoration` — `Network.add_justification` restores an OUT node to IN when a valid new justification is added, as tested in `tests/test_add_justification.py:43-60`.
- `explicit-restoration-clears-retracted-flag` — `reasons/api.py:1637` pops `_retracted` before calling `net.recompute_all()` at `reasons/api.py:1647`.
- `import-agent-clears-retracted-on-remote-in` — `reasons/import_agent.py:425-426` and `485` clear `_retracted` when the remote source reports IN.
