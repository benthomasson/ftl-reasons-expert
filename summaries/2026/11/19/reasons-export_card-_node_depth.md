---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: _node_depth in reasons/export_card.py

## `_node_depth` — Explanation

### Purpose

`_node_depth` computes the **maximum derivation depth** of a single belief node within a justificatory network. In a Reasons/EEM-style network, "premises" are nodes with no justifications (grounded observations), while "derived" nodes are backed by one or more justifications, each of which references *antecedent* nodes. The depth of a node is:

- `0` if it is a premise (or missing);
- `1 + max(depth of each antecedent)` if it is derived.

It exists so `export_card` can report a single `Max derivation depth` statistic across the whole network (used in the generated README table), giving a coarse measure of how many inference layers the network contains.

### Contract

- **Preconditions**: `nid` is a key (or not) in `nodes`. Each `node` in `nodes` exposes `.justifications` (sequence) and `.supporting_justification` (index-like or `None`). Each justification exposes `.antecedents` (iterable of ids). No assumption is enforced that antecedent ids are present in `nodes`.
- **Postconditions**: Returns a non-negative integer. `memo[nid]` is populated with the returned value. The value equals `0` for missing/premise nodes and `1 + max(antecedent depths)` otherwise (subject to the cycle caveat below).
- **Invariants**:
  - Memoized results are reused for repeated calls with the same `memo`.
  - `memo[nid]` is set to `0` *before* recursion, so cycles do not infinitely recurse — they terminate with depth `0` at the point of revisit.

### Parameters

| Param | Type | Meaning | Edge cases |
|---|---|---|---|
| `nid` | hashable (typically `str`/`int` node id) | Node whose depth is wanted | Missing id → returns `0` |
| `nodes` | mapping with `.get(nid)` (passed `network.nodes`) | Node lookup table | Any mapping honoring `.get` works |
| `memo` | `dict[int, int] | None` | Cache of computed depths | If `None`, a fresh dict is allocated. If supplied, it is **mutated** and shared across calls |

The code assumes `nodes` is dict-like (uses `.get`, but *not* `__getitem__`), and that antecedent ids are hashable. It also assumes `supporting_justification` is comparable to integers (`0 <= si`) — a non-numeric, non-`None` value would raise `TypeError`.

### Return Value

Returns a non-negative `int`:
- `0` — node missing, or node has no `justifications` (premise).
- `max_d + 1` — for a node with at least one justification whose antecedents' depths are `max_d`.
- `0` at a cycle revisit (memo pre-set), which can produce **underestimates** for cyclic graphs and, because those `0`s are cached, an order-dependent result if the same `memo` is reused across roots in a cyclic graph.

Callers must handle a plain integer; no exceptions are deliberately raised or returned.

### Algorithm (step-by-step)

1. **Init memo**: if `memo is None`, allocate `{}`.
2. **Memo hit**: if `nid in memo`, return `memo[nid]`.
3. **Base case**: `node = nodes.get(nid)`. If `node` is falsy or `node.justifications` is empty → `memo[nid] = 0`; return `0`.
4. **Cycle guard**: set `memo[nid] = 0` **before** recursing so that any self-reference encountered during recursion short-circuits to `0` instead of recursing infinitely.
5. **Justification selection**:
   - `si = node.supporting_justification`.
   - If `si` is not `None` and satisfies `0 <= si < len(node.justifications)`, use **only** `[node.justifications[si]]`.
   - Otherwise, use **all** `node.justifications`. This means an invalid supporting index silently falls back to "use everything" rather than erroring.
6. **Recurse**: for each justification `j` in the selected list, for each antecedent `a` in `j.antecedents`, call `_node_depth(a, nodes, memo)` and take the max.
7. **Write result**: `memo[nid] = max_d + 1`; return `max_d + 1` (overwriting the placeholder `0` set in step 4).

The shared `memo` in `export_card`'s `max(...)` comprehension makes the whole network traversal roughly `O(V + E)` on a DAG.

### Side Effects

- Mutates the caller-supplied `memo` dict (adds keys).
- Recursive calls; subject to Python's recursion limit for very deep acyclic chains (~1000+).
- No I/O, no global state, no exceptions raised intentionally.

### Error Handling

- **No `try/except`**. Errors propagate:
  - `AttributeError` if a node lacks `.justifications` / `.supporting_justification`, or a justification lacks `.antecedents`.
  - `TypeError` if `supporting_justification` is not `None` and not comparable to `int` (e.g., a string).
  - `RecursionError` for pathologically deep acyclic chains.
- **Invalid supporting index is silently ignored** (falls back to all justifications) rather than raising — a deliberate-looking leniency that hides data inconsistencies.
- **Missing antecedent ids are silently treated as depth 0** via `nodes.get(nid) → None`.

### Usage Patterns

Called only from `export_card`:

```python
memo = {}
max_depth = max(
    (_node_depth(nid, network.nodes, memo) for nid in network.nodes),
    default=0,
)
```

Callers should:
- Pass a **fresh** `memo` per export to avoid cross-contamination (though order-dependence only matters for cyclic graphs).
- Ensure all nodes are reachable / present in `network.nodes` — otherwise missing ids silently return `0`, which can mask bugs.
- Be aware that the returned depth is only meaningful for **acyclic** justification graphs; for cyclic graphs the value is a lower bound due to the pre-set `0`.

### Dependencies

- No imports within the function.
- Relies on the structural shape of `Network.nodes` values: `.justifications`, `.supporting_justification`, and justification `.antecedents`.
- Imported alongside `Network` in `reasons/export_card.py`; used only by `export_card`.

### Assumptions not enforced by the type system

- `nodes` supports `.get` but is not required to be a `dict`.
- Antecedent tuples contain ids that are keys in `nodes`; missing ids silently degrade to `0`.
- `supporting_justification` is `None` or an `int`-comparable; a string would crash on `0 <= si`.
- Justifications/antecedents are iterable; a `None` `.antecedents` would raise `TypeError` in the `for` loop.
- The graph is expected to be (mostly) acyclic; cycles are tolerated but undercount.

## Topics to Explore

- [function] `reasons/export_card.py:export_card` — The only consumer of `_node_depth`; see how `max_depth` feeds the generated README table and stats.
- [file] `reasons/network.py` — Defines `Network.nodes` and the `Node`/justification shapes (`justifications`, `supporting_justification`, `antecedents`) the function depends on.
- [general] `cycle-handling-in-recursive-memoization` — Why setting `memo[nid] = 0` before recursing terminates cycles but can produce order-dependent depths.
- [file] `tests/test_export_card.py` — Existing tests; check whether cyclic graphs or invalid `supporting_justification` indices are covered.
- [general] `supporting-justification-semantics` — What `supporting_justification` means in Reasons and when it is `None` vs. an out-of-range index.

## Beliefs

- `node-depth-base-case` — A node that is missing from `nodes` or has an empty `justifications` list returns depth `0`.
- `node-depth-derivation-formula` — For a node with justifications, the returned depth is `1 + max(depth(antecedent))` over the selected justifications.
- `supporting-justification-narrows-traversal` — When `supporting_justification` is a valid index, only that single justification's antecedents are traversed; otherwise all justifications are.
- `memo-pre-set-prevents-infinite-recursion` — `memo[nid]` is set to `0` before recursing, which terminates cycles but can undercount depth in cyclic graphs.
- `missing-antecedent-yields-zero` — An antecedent id absent from `nodes` is silently scored as depth `0` rather than raising.
- `export-card-requires-node-depth` — `export_card` depends on `_node_depth` to populate the `Max derivation depth` metric in the generated card.
