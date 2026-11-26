---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Search for tests or call paths that invoke depth helpers with multiple starting nodes or cycles.

## What the observations establish

The depth helpers that were found are single-entry recursive functions, not multi-start APIs:

- `reasons/api.py:3368` defines `_node_depth(nid, net, memo=None)`.
- `reasons/derive.py:114` defines `_get_depth(node_id, nodes, derived, memo=None)`.
- `reasons/export_card.py:7` defines `_node_depth(nid, nodes, memo=None)`.
- `reasons/repair.py:270` defines `_compute_depth(node_id, nodes, memo=None)`.

All observed signatures take one starting node id. If multiple starting nodes are supported, it would have to happen at a call site by looping over a collection and calling the helper repeatedly; the observations do not show such a call path.

## Cycle-related evidence

Two helpers have visible cycle guards:

- `reasons/api.py:3368 _node_depth` sets `memo[nid] = 0  # cycle guard` at `reasons/api.py:3378`.
- `reasons/derive.py:114 _get_depth` sets `memo[node_id] = 0  # cycle guard` at `reasons/derive.py:123`.

No cycle guard is visible in the observed lines for `reasons/export_card.py:7 _node_depth`, `reasons/repair.py:270 _compute_depth`, `reasons/api.py:737 _cascade_depth`, or `reasons/pg.py:559 _cascade_depth_pg`, but the observations only include definitions, not full bodies, so we cannot conclude those helpers lack cycle handling.

The cycle search did find `tests/test_merkle.py:193 test_cycle_safety`, but that test is explicitly about merkle antecedent-cycle safety (`tests/test_merkle.py:194`), not about `_node_depth`, `_get_depth`, `_compute_depth`, or prompt depth expansion.

## Test coverage found by name

Several tests mention depth, but the observations do not include their bodies:

- `tests/test_derive.py:109 test_get_depth` is the most likely direct unit test for `_get_depth`.
- `tests/test_api.py:385`, `391`, and `400` test API depth expansion behavior: depth 1, depth 2, and depth 0.
- `tests/test_derive.py:123`, `133`, and `144` test prompt depth filters.
- `tests/test_export_card.py:179` tests premises-only depth zero.
- `tests/test_research.py:144`, `148`, and `181` test research prompt/depth behavior.
- `tests/test_review.py:343` tests a min-depth filter.

However, the observation `find_depth_usages_in_tests` reports zero matches for pattern `depth`, which contradicts the earlier matches showing test names containing `depth`. That makes the usage-search result unreliable or scoped differently than the definition search. We therefore cannot infer actual call arguments or whether these tests exercise cycles or multiple starting nodes.

## Multiple-starting-node evidence

The multiple-starting-node search found:

- `tests/test_check_stale.py:363 test_multiple_nodes`
- `tests/test_check_stale_issue25.py:92 test_multiple_nodes_same_deleted_source`
- `tests/test_dangling_dependents.py:165 test_same_dangling_in_multiple_nodes`
- `tests/test_outlist.py:118 test_multiple_outlist_nodes`

None of these are depth-helper tests based on their names/locations. The term search also found `reasons/import_agent.py:3`, but it describes namespaced nodes for multiple agents, not depth traversal.

## Conclusion

Based on the provided observations, there is **no confirmed test or call path that invokes a depth helper with multiple starting nodes or a cyclic graph**. The only observable cycle handling is inside `reasons/api.py:_node_depth` and `reasons/derive.py:_get_depth`; the only observable cycle test is for merkle verification, not depth helpers. To answer definitively, we need the missing call-site search results and the bodies of the depth-related tests, especially `tests/test_derive.py:test_get_depth` and the API depth tests.

## Topics to Explore

- [function] `reasons/derive.py:_get_depth` — Inspect the full recursion and memo behavior to confirm how cycles are handled.
- [function] `reasons/api.py:_node_depth` — Compare its cycle guard with API depth-expansion call sites.
- [file] `tests/test_derive.py` — Read `test_get_depth` and prompt-depth tests to see whether they pass one node or multiple starting nodes.
- [file] `tests/test_merkle.py` — Confirm `test_cycle_safety` covers merkle verification only, not depth helpers.
- [general] `depth-helper-call-sites` — Search the whole repo for invocations of `_node_depth`, `_get_depth`, `_compute_depth`, `_cascade_depth`, and `_cascade_depth_pg` to find loops over multiple starts or cyclic test graphs.

## Beliefs

- `depth-helpers-take-single-node-id` — The observed depth helper signatures accept one node identifier, not a collection of starting nodes.
- `api-and-derive-depth-have-cycle-guards` — `reasons/api.py:_node_depth` and `reasons/derive.py:_get_depth` contain explicit memo-based cycle guards at lines 3378 and 123.
- `cycle-safety-test-does-not-target-depth-helpers` — `tests/test_merkle.py:test_cycle_safety` verifies merkle antecedent-cycle behavior, not depth-helper recursion.
- `observed-multiple-node-tests-are-not-depth-tests` — The multiple-node tests found in stale, dangling, and outlist test files do not target the depth helpers.
- `depth-usage-search-is-inconclusive` — The reported zero matches for depth usage in tests conflicts with earlier test-name matches, so call-path coverage cannot be inferred from that search alone.
