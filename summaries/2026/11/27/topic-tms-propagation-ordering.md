---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Whether the engine does a single topological pass or iterates to a fixpoint; matters for cycles in outlist↔antecedent dependencies.

# Single pass vs. fixpoint iteration

## Short answer

**The core truth-value propagation iterates to a fixpoint**, not a single topological pass. The iteration is bounded by `len(self.nodes) + 1` and each pass uses a queue-based worklist with a `visited` cycle guard. Topological sorts exist in the codebase, but they are restricted to **import ordering**, not belief propagation.

## Evidence from the code

### The main propagation loop is an explicit fixpoint

In `reasons/network.py`:

- Line 907 is a literal docstring fragment: `"Iterates to a fixpoint so cascading changes propagate regardless of"`.
- Line 911: `max_iterations = len(self.nodes) + 1`.
- Line 912: `for _ in range(max_iterations):` — an **outer iteration cap**.
- Lines 930–956: inside each iteration, a queue-based BFS (`queue = deque([changed_id])`, `while queue:`, `queue.append(dep_id)`) walks the dependents graph.

So the shape is *outer fixpoint loop → inner worklist BFS*. That is exactly the anti-pattern a topological sort replaces, and it is done on purpose because cycles may exist.

### Both edge kinds are in the dependents index

`reasons/network.py:_rebuild_dependents` (around lines 34–47), and the mirrored logic in `add_node` (~line 155), register dependents from **both** `j.antecedents` and `j.outlist`:

```python
for ant_id in j.antecedents: ...
for out_id in j.outlist: ...
```

This is critical for the question: an outlist-only reference (e.g. `X unless Y`) still makes `X` a dependent of `Y`. That means asserting/retracting `Y` triggers propagation into `X`. Combined with the fixpoint loop, an `outlist ↔ antecedent` cycle (e.g. `A` in `B`'s antecedents and `A` in `B`'s outlist via a chain, or `A` outlist-depends on `B` which antecedent-depends on `A`) is *structurally allowed* and will be repeatedly re-evaluated.

### Cycles don't crash, but they don't necessarily converge cleanly

- Within a single pass, cycle guards exist: `visited = {changed_id}` (network.py:221, 226, 233) and `seen`/`visited` sets (network.py:246, 271, 453). These prevent infinite BFS *within* one pass.
- The *outer* loop's `max_iterations = len(self.nodes) + 1` is the real termination guarantee. If a cycle oscillates truth values every pass (e.g. `A unless B` and `B unless A`), the loop simply exhausts its budget and stops — the observation set doesn't show any explicit "if not changed: break" convergence check for the truth-value pass, so the practical guarantee is *bounded execution*, not necessarily *convergence to a stable assignment*.

### Where topological sorts *do* appear

- `reasons/import_agent.py:191` — `_topo_sort_claims(claims)`, used at lines 273 and 354 to order claims before inserting them.
- `reasons/import_beliefs.py:201` — `# Iterative topological sort (handles missing deps gracefully)`.

These are **pipelines for ordering input**, not the runtime truth engine. They do not handle outlist↔antecedent cycles — they only know about import-time claim dependencies.

### Depth computation uses a cycle guard, not a fixpoint

`reasons/derive.py:_get_depth` (line 122–124) does `memo[node_id] = 0  # cycle guard` before recursing, and the same pattern appears at `reasons/api.py:3378`. This means a cyclic subgraph yields a *bounded but arithmetically wrong* depth (it terminates a cycle at depth 0 for the node that closes it). Depth is a display/compute heuristic, not part of propagation.

## What this means for outlist ↔ antecedent cycles

1. You can *create* such a cycle without an error — `add_node` computes initial truth via `_compute_truth`, and the fixpoint handles subsequent changes.
2. Truth values in a cycle **may oscillate across outer passes**; the `len(self.nodes) + 1` cap prevents an infinite loop, but there is no visible "we did not reach a fixpoint" warning in the observations.
3. The `visited` guards inside `_propagate_access_tags` (network.py:220–234) apply to access-tag propagation, not to truth-value propagation — they won't save you from a truth-value oscillation.
4. If you genuinely need "eventual semantics" (like a well-founded model), you would need either stratification of outlist edges or an explicit oscillation detector — neither is visible here.

## What's missing from the observations

- The body of the fixpoint function at network.py:907–956 is not fully shown (the read returned only 300 of 1002 lines and cut off just before this function). We can see the *shape* but not whether there is a `changed` flag or explicit `break`-on-convergence.
- `_compute_truth` (called from `add_node`, referenced in the fixpoint) is not shown — this is what actually evaluates `justifications` against antecedents and outlist per pass, and it determines whether cycles can break or stay in oscillation.
- No test in the supplied snippets exercises an actual outlist↔antecedent cycle (tests/test_outlist.py covers chains and alternate justifications, but not cycles).

## Topics to Explore

- [function] `reasons/network.py:_propagate` — The fixpoint function (around lines 900–960); confirm whether there is an explicit convergence check or only the `len(nodes)+1` budget.
- [function] `reasons/network.py:_compute_truth` — Per-node evaluator for SL/CP + outlist; reveals whether a cyclic node is "short-circuited" to OUT or reevaluated freely.
- [file] `reasons/derive.py` — See `_get_depth` (line 122) and `_build_derived_section`; depth is cycle-truncated, which affects how derived cycles render in prompts.
- [file] `reasons/import_agent.py` — `_topo_sort_claims` (line 191) is the only real topo sort in the pipeline; contrast its semantics with the fixpoint engine.
- [general] `outlist-antecedent-cycle-behavior` — Add a targeted test with `A unless B; B sl=A` and observe whether truth stabilizes, oscillates, or hits `max_iterations`.

## Beliefs

- `network-propagation-iterates-to-fixpoint` — Truth-value propagation runs an outer `for _ in range(len(self.nodes)+1)` fixpoint loop around an inner worklist BFS, not a single topological pass (`reasons/network.py:907-956`).
- `dependents-index-includes-outlist-edges` — `_rebuild_dependents` and `add_node` register dependents for both `antecedents` and `outlist` references, so outlist-only edges trigger propagation (`reasons/network.py:34-47`, ~155).
- `depth-computation-cycle-guard-truncates` — `_get_depth` in `reasons/derive.py:122-124` and the parallel code at `reasons/api.py:3378` memoize `0` on cycle entry, so cyclic depth values are bounded but not semantically meaningful.
- `topological-sort-is-confined-to-import` — The only topological sort routines are `_topo_sort_claims` in `reasons/import_agent.py:191` and the inline sort at `reasons/import_beliefs.py:201`; neither is part of the runtime truth engine.
- `access-tag-propagation-is-single-bfs-pass` — `_propagate_access_tags` (`reasons/network.py:220-234`) uses a single `visited`-guarded BFS with no outer fixpoint, unlike the truth-value pass.
