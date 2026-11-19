---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: How `supporting_justification` is kept valid across add/remove operations (see `tests/test_add_justification.py` and `tests/test_remove_justification.py`); this is the most likely source of latent bugs in the model.

## Short answer

Based on the supplied observations, it is **not possible to verify how `supporting_justification` is kept valid across add/remove operations**. The identifier `supporting_justification` has **zero matches** in both code and tests:

- `find_supporting_justification_usages`: `total_count: 0`
- `find_supporting_justification_in_tests`: `total_count: 0`

The add/remove tests never mention `supporting_justification`. They only validate observable outcomes such as `truth_value`, `justifications` length/content, `dependents`, `changed`, and operation results. Therefore, if `supporting_justification` is a cached pointer or derived field, the tests as observed do **not** directly protect it from becoming stale.

The code locations we do have are:

- `reasons/api.py:406` — `add_justification`
- `reasons/api.py:1129` — `remove_justification`
- `reasons/mcp_server.py:204` — MCP wrapper `add_justification`

The tests call `Network.add_justification` and `Network.remove_justification`, but the observations do **not** include the `Network` implementation or the `Node` model where `supporting_justification` would likely live.

## What the tests actually establish

### Add operations

`tests/test_add_justification.py` requires:

- Adding a second justification to an `IN` node keeps it `IN` and increases `justifications` length (`test_add_justification_to_derived_node`).
- A second justification keeps the node `IN` after one antecedent is retracted (`test_added_justification_keeps_node_in_after_retract`).
- Adding a valid justification to an `OUT` node flips it to `IN` and records it in `changed` (`test_add_justification_restores_out_node`).
- Restoring a node cascades to dependents (`test_add_justification_cascades`).
- Adding a justification with an unsatisfied `outlist` does **not** immediately make the node `IN`; once the outlist node is retracted, the node becomes `IN` (`test_add_justification_with_outlist`).
- Premises can receive backup justifications (`test_add_justification_to_premise`).
- Antecedent and outlist nodes get the target node registered as a dependent (`test_add_justification_registers_dependents`).
- API-level add works for `SL`, `CP`, `unless`, namespaces, and restoring an `OUT` node (`TestAPIAddJustification`).

### Remove operations

`tests/test_remove_justification.py` requires:

- Removing one of two justifications leaves the other intact (`test_remove_one_of_two`).
- Removing the active justification can flip `IN` to `OUT` (`test_remove_causes_out`).
- Truth-value changes cascade to dependents (`test_remove_propagates_cascade`).
- Dependents are cleaned only when no remaining justification still references the node (`test_remove_cleans_dependents`, `test_remove_keeps_shared_dependent`).
- Outlist dependencies are also cleaned (`test_remove_outlist_cleans_dependents`).
- Removing a premise’s justification, removing the only justification, using a bad index, or removing from a missing node raises errors (`test_error_on_premise`, `test_error_on_single_justification`, `test_error_on_bad_index`, `test_error_on_missing_node`).
- API-level removal persists and updates the node (`TestApiRemoveJustification`).

## Why this is a likely latent-bug hotspot

If `supporting_justification` is a cache of “which justification currently supports this node”, then add/remove correctness is much stronger than what the observed tests verify.

For example:

- On **add**, the implementation must not simply set `supporting_justification` to the newly added justification. `test_add_justification_with_outlist` proves an added justification can be invalid because an outlist node is currently `IN`.
- On **remove**, the implementation must not leave `supporting_justification` pointing at the removed justification or at an invalid remaining justification. `test_remove_causes_out` proves that after removing one justification the node can become `OUT`.
- Removing an index shifts the `justifications` list, so any cache keyed by index can go stale. The observed tests check labels after removal (`test_remove_one_of_two`) but not whether a cached `supporting_justification` still references the correct object.
- Dependents cleanup has similar conditional semantics: a dependent must remain registered if another justification still references that antecedent or outlist node (`test_remove_keeps_shared_dependent`, `test_remove_outlist_cleans_dependents`). A parallel cache could easily use simpler, incorrect rules.

So the tests are good at checking externally visible truth values and dependency registration, but without direct assertions on `supporting_justification`, a stale-cache bug could pass all observed tests.

## What is missing

To explain the actual mechanism, we need:

- `reasons/network.py`, including `Network.add_justification`, `Network.remove_justification`, and the `Node` class.
- Any property/field definition or serialization path for `supporting_justification`.
- The truth-value recomputation logic used after add/remove.
- Tests that assert `supporting_justification` identity/content before and after add/remove/retract/outlist cases.

## Topics to Explore

- [file] `reasons/network.py` — Contains `Network` and likely the `Node` model, truth propagation, dependent registration, and the actual add/remove implementations called by the tests.
- [function] `reasons/api.py:add_justification` — API entry point at `reasons/api.py:406`; coordinates persistence and network mutation for added justifications.
- [function] `reasons/api.py:remove_justification` — API entry point at `reasons/api.py:1129`; coordinates persistence, deletion, and cascade behavior.
- [file] `tests/test_add_justification.py` — Add direct assertions for `supporting_justification` around valid adds, invalid outlist adds, premises, and cascades.
- [file] `tests/test_remove_justification.py` — Add direct assertions for `supporting_justification` around active-justification removal, index shifts, shared dependents, and outlist cleanup.

## Beliefs

- `supporting-justification-has-zero-observed-usages` — The search for `supporting_justification` returned no matches in code or tests.
- `add-justification-api-defined-at-api-py-406` — `add_justification` is defined at `reasons/api.py:406` and also at `reasons/mcp_server.py:204`.
- `remove-justification-api-defined-at-api-py-1129` — `remove_justification` is defined at `reasons/api.py:1129`.
- `network-justification-methods-not-observed` — The tests call `Network.add_justification` and `Network.remove_justification`, but their definitions are absent from the supplied observations.
- `justification-tests-assert-truth-and-dependents-not-cache` — Add/remove tests assert truth values, justification counts, dependents, and changed sets, but none assert `supporting_justification`.
