---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Why setting `memo[nid] = 0` before recursing terminates cycles but can produce order-dependent depths.

The depth helpers use one `memo` dict keyed only by node id. The pattern is visible in four places:

- `reasons/api.py:_node_depth` — line 3368, with memo hit at 3373, zero assignments at 3376/3378, final assignment at 3388.
- `reasons/derive.py:_get_depth` — line 114, with memo hit at 119, zero assignments at 121/123, final assignment at 135.
- `reasons/export_card.py:_node_depth` — line 7, with memo hit at 11, zero assignments at 14/16, final assignment at 26.
- `reasons/repair.py:_compute_depth` — line 270, with memo hit at 275, zero assignments at 278/280, final assignment at 285.

Conceptually the function is:

```python
def depth(nid, ..., memo=None):
    if nid in memo:
        return memo[nid]

    # base/missing case
    memo[nid] = 0
    ...

    memo[nid] = 0          # cycle guard
    max_d = 0
    for antecedent in ...:
        max_d = max(max_d, depth(antecedent, ..., memo))

    memo[nid] = max_d + 1
    return memo[nid]
```

The cycle guard works because recursion checks `memo` first. Before recursing into antecedents, the current node is written as `0`. If the recursion later reaches the same node through a cycle, the memo hit returns `0` instead of recursing again. Without that early write, `A -> B -> A` would recurse forever because `A` would not be memoized until after its children finished.

The problem is that `0` is overloaded. It means all of these:

- a real base case / premise depth,
- possibly a missing node depth,
- and “this node is currently being visited.”

The final value is overwritten later with `memo[nid] = max_d + 1` (`api.py:3388`, `derive.py:135`, `export_card.py:26`, `repair.py:285`), but during recursion the placeholder `0` is observable.

That makes the result order-dependent on cyclic graphs. For example, suppose:

- `A` depends on `B` and premise `P`
- `B` depends on `A` and premise `Q`

Starting at `A`:

1. set `memo[A] = 0`
2. recurse into `B`
3. `B` recurses into `A`, gets `memo[A] == 0`
4. `B` becomes `1`
5. `A` becomes `max(1, 0) + 1 == 2`

So `A=2`, `B=1`.

Starting at `B` gives the reverse:

1. set `memo[B] = 0`
2. recurse into `A`
3. `A` recurses into `B`, gets `memo[B] == 0`
4. `A` becomes `1`
5. `B` becomes `max(1, 0) + 1 == 2`

So `A=1`, `B=2`.

The cycle is terminated, but the cycle is effectively “cut” at whichever back-edge is encountered first. Because `memo` is keyed only by node id and is reused across calls, a depth computed under one traversal order is then reused for later roots. The observations do not include the full antecedent iteration order or every base case, so the exact value for a specific graph depends on those unseen details, but the line pattern clearly shows the visiting marker is being stored as `0` and later overwritten.

## Topics to Explore

- [function] `reasons/derive.py:_get_depth` — Inspect the full body to see the antecedent iteration order and exact base cases; those determine which cycle edge is cut first.
- [function] `reasons/api.py:_node_depth` — Compare the duplicated implementation and check whether callers pass a shared `memo` across roots.
- [file] `tests/test_derive.py` — Look at depth tests such as `test_get_depth` to see whether cyclic networks are covered.
- [general] `cycle-depth-order-dependence` — Search for tests or call paths that invoke depth helpers with multiple starting nodes or cycles.
- [file] `reasons/repair.py` — Check whether `_compute_depth` is used on graphs that can contain cycles, since it repeats the same guard pattern.

## Beliefs

- `cycle-guard-uses-zero-placeholder` — The depth helpers set `memo[nid] = 0` before recursing to mark an in-progress node.
- `depth-functions-overwrite-placeholder-after-recursion` — After visiting antecedents, the helpers overwrite the placeholder with `max_d + 1`.
- `depth-memo-is-keyed-only-by-node-id` — The memo cache does not distinguish “visiting” from “finished” and is not parameterized by root or traversal order.
- `depth-order-dependence-requires-cycles` — In an acyclic graph, memoized max-depth should be order-independent; order-dependence appears when the zero placeholder is read through a cycle.
- `depth-code-duplicated-across-files` — The same memoized depth pattern appears in `api.py`, `derive.py`, `export_card.py`, and `repair.py`.
