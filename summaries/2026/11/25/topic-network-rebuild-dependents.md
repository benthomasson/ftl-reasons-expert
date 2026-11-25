---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Why `_fixup_dependents`/`_rebuild_dependents` exists: outlist edges added after node creation require explicit dependent re-registration or revocation won't actually cascade.

The reverse dependency index (`Node.dependents`) is not recomputed during propagation. It is a stored index that `_propagate` walks when a node changes truth value. If a node is missing from that index, changes to its “parent” will not reach it.

For SL justifications, both antecedents and `outlist` entries are dependency edges:

- `antecedents` must be **IN**
- `outlist` entries must be **OUT**

So if `x` has a justification `SL(antecedents=[], outlist=["y"])`, then `y` must know that `x` depends on it. `tests/test_dependents_integrity.py` demonstrates this directly: after `_rebuild_dependents()`, an outlist reference causes the outlist node’s `dependents` set to contain the dependent node (`test_outlist_dependency`, around lines 53-60), and both antecedent and outlist edges are registered together (`test_both_antecedent_and_outlist`, around lines 62-70).

That matters because propagation is driven by `dependents`. `tests/test_dangling_dependents.py` shows the mechanism: `retract("a")` walks `a.dependents`; valid dependents are updated, and dangling entries are warned about and skipped. If a real outlist dependent is missing from `a.dependents`, then asserting or retracting `a` will not cascade to the node whose justification has `outlist=["a"]`. The dependent would incorrectly remain `IN` or `OUT`.

Normal mutation paths are supposed to maintain this index. For example, `tests/test_dependents_integrity.py::test_after_add_justification` verifies that `add_justification("c", ... antecedents=["b"])` adds `"c"` to `b.dependents`. But import code can create or modify nodes/justifications in phases, so an outlist edge may be present in the justification data without the reverse index having been updated. That is exactly the import-specific repair point:

- `reasons/import_agent.py:31` defines `_fixup_dependents(network)`.
- `reasons/import_agent.py:37` calls `network._rebuild_dependents()` inside that fixup.
- `reasons/import_agent.py:316` and `reasons/import_agent.py:481` call `_fixup_dependents(network)` after import work.
- `reasons/network.py:27` defines the canonical `_rebuild_dependents()` implementation.
- `reasons/storage.py:433` also calls `network._rebuild_dependents()`, consistent with rebuilding the reverse index after loading persisted state.

`_rebuild_dependents()` exists because the safe repair is to clear the reverse index and reconstruct it from the actual justifications. `tests/test_dependents_integrity.py::test_rebuild_clears_stale_entries` confirms that it removes stale entries before rebuilding, and the outlist test confirms that outlist edges are included in the reconstruction. Without that repair, import would produce a network where the justifications look correct but revocation/restoration does not cascade through outlist edges, because the stored `dependents` index is stale.

The exact bodies of `_fixup_dependents` and `_rebuild_dependents` are not fully shown in the observations, but the tests establish the behavior clearly enough: the reverse index must include outlist edges, propagation depends on it, and import explicitly rebuilds it.

## Topics to Explore

- [function] `reasons/import_agent.py:_fixup_dependents` — Read the full wrapper to see whether it only rebuilds dependents or performs extra import-time validation.
- [function] `reasons/network.py:_rebuild_dependents` — Inspect how the canonical rebuild clears `dependents` and scans both antecedents and outlist entries.
- [file] `reasons/network.py` — Look at `_propagate`, `add_node`, `add_justification`, `retract`, and `assert_node` to see where the reverse index is normally maintained.
- [file] `tests/test_outlist.py` — The cascading tests show the user-visible failure mode when outlist dependents are not registered.
- [file] `tests/test_dependents_integrity.py` — Covers mutation-path integrity, stale-entry cleanup, and `verify_dependents()` / `_rebuild_dependents()` repair behavior.

## Beliefs

- `dependents-index-includes-outlist-edges` — `_rebuild_dependents()` registers outlist references as reverse dependencies, as shown by `test_outlist_dependency` and `test_both_antecedent_and_outlist` in `tests/test_dependents_integrity.py`.
- `propagation-requires-dependents-index` — Truth-value propagation only reaches nodes present in the changed node’s `dependents` set; dangling-dependent tests demonstrate that `retract` and `assert_node` walk that set.
- `import-agent-requires-fixup-dependents` — `reasons/import_agent.py` calls `_fixup_dependents(network)` at lines 316 and 481, and `_fixup_dependents` invokes `network._rebuild_dependents()` at line 37.
- `rebuild-dependents-clears-stale-edges` — `_rebuild_dependents()` removes stale entries before rebuilding, per `test_rebuild_clears_stale_entries` in `tests/test_dependents_integrity.py`.
- `storage-load-requires-rebuild-dependents` — `reasons/storage.py:433` calls `network._rebuild_dependents()`, indicating persisted networks need reverse-index reconstruction after loading.
