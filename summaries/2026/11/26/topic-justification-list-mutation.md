---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Search for code that removes or reorders `node.justifications`; that is the likely way a stored index becomes out of range.

## What the search found

The strongest direct hit for removing entries from `node.justifications` is:

- `reasons/network.py:727` — `removed = node.justifications.pop(index)`

This is inside the network-level `remove_justification` implementation, which is called by the API wrapper:

- `reasons/api.py:1129` — `def remove_justification(...)`
- `reasons/api.py:1138` — `return net.remove_justification(node_id, index)`

The mutation search did **not** find `sort`, `reverse`, `insert`, `clear`, or `del` operations on `justifications` in the gathered matches. So within these observations, the main removal mechanism is `pop(index)`, which shortens the list and shifts every later justification down by one index.

There is also one full-list reassignment worth inspecting:

- `reasons/import_agent.py:83` — `network.nodes[node_id].justifications = new_justifications`

That can replace the entire justification list and may reorder or drop entries depending on how `new_justifications` is built.

## Why a stored index can become out of range

`supporting_justification` is explicitly documented as an index into the justification list:

- `reasons/__init__.py:37` — `supporting_justification: int | None = None  # index into justifications list`

The defeat-related APIs also accept, validate, return, and embed a `justification_index`:

- `reasons/api.py:1363` — checks `justification_index < 0 or justification_index >= len(node.justifications)`
- `reasons/api.py:1365` — raises `"Justification index ... out of range"`
- `reasons/api.py:1369` — indexes `node.justifications[justification_index]`
- `reasons/api.py:1372` — creates an ID such as `defeater-...-j{justification_index}`
- `reasons/api.py:1381` — returns `"defeats_justification": justification_index`
- `reasons/api.py:1461-1467` — the same pattern appears again for another defeat path
- `reasons/api.py:1508` — returns `"defeats_justification": justification_index`

So if an index such as `defeats_justification` or `supporting_justification` is stored and then someone calls `remove_justification` on a lower index, the old stored index will now refer to a different justification or may exceed the new list length.

The tests confirm the index-shifting behavior:

- `tests/test_remove_justification.py::TestNetworkRemoveJustification::test_remove_one_of_two` removes index `0`, then asserts that the former second justification is now at index `0`.

That is exactly the behavior that can invalidate any previously stored index.

There is one defensive read path:

- `reasons/api.py:3380` — checks `0 <= si < len(node.justifications)` before indexing
- `reasons/api.py:3381` — uses `[node.justifications[si]]` only when in range

That prevents an immediate `IndexError` in that one path, but it does not solve semantic staleness: the index may still point to the wrong justification after a removal.

## Likely source of the out-of-range index

Based on the observations, the likely sequence is:

1. A caller or defeater stores a `justification_index` or `supporting_justification`.
2. `remove_justification` later runs and executes `node.justifications.pop(index)` at `reasons/network.py:727`.
3. Remaining justifications shift down, and the list gets shorter.
4. Later code reads the old stored index, for example at `reasons/api.py:1369` or `reasons/api.py:1467`.
5. The range check at `reasons/api.py:1363` or `1461` raises an out-of-range error.

The gathered matches do not show any compensating code that rewrites stored `supporting_justification` or `defeats_justification` values after a removal. That is worth verifying in the full implementation, because the search results alone do not prove there is no such repair logic elsewhere.

## Caveat

These observations are search hits rather than a full call graph. The absence of `sort`/`reverse`/`clear` in the mutation search does not prove reordering never happens. In particular, full-list replacement at `reasons/import_agent.py:83` should be inspected, and aliases such as `justifications = node.justifications` in export/show paths should be checked to confirm they are not later mutated.

## Topics to Explore

- [function] `reasons/network.py:Network.remove_justification` — Inspect the full method around line 727 to confirm the `pop(index)` behavior, dependents cleanup, propagation, and whether it updates any stored justification indexes.
- [function] `reasons/api.py:remove_justification` — Read the API wrapper at lines 1129-1138 to see how CLI/API removal reaches the network mutation and what results are persisted or returned.
- [file] `reasons/import_agent.py` — Inspect the whole-list assignment at line 83 to determine whether imported justifications can be reordered or filtered.
- [general] `justification-index-staleness` — Trace all read/write sites of `supporting_justification`, `justification_index`, and `defeats_justification` to find where stale indexes could be persisted or reused.
- [file] `tests/test_remove_justification.py` — The tests verify post-pop index shifting but do not appear to cover a stored-index regression after removal.

## Beliefs

- `remove-justification-uses-pop` — `Network.remove_justification` removes a justification with `node.justifications.pop(index)` at `reasons/network.py:727`.
- `api-remove-justification-delegates-to-network` — `api.remove_justification` delegates to `net.remove_justification(node_id, index)` at `reasons/api.py:1138`.
- `supporting-justification-is-an-index` — `supporting_justification` is documented as an integer index into the justifications list at `reasons/__init__.py:37`.
- `defeat-flow-returns-justification-index` — Defeat code paths return `"defeats_justification": justification_index` and embed `j{justification_index}` in defeater IDs at `reasons/api.py:1372-1381` and `1470-1508`.
- `remove-justification-shifts-indexes` — Removing index `0` from a two-justification list makes the former index `1` become index `0`, as asserted in `tests/test_remove_justification.py::TestNetworkRemoveJustification::test_remove_one_of_two`.
