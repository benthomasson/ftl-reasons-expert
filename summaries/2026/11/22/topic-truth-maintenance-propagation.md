---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: The full recompute/fixpoint algorithm is not shown in these observations; inspect it to confirm ordering and multiple-justification behavior.

## Explanation

### What the observations actually show

The recommendation to "inspect it to confirm ordering and multiple-justification behavior" is correct — the described recompute/fixpoint algorithm is **not** present in these observations. What we have are entry points, docstrings, tests, and surrounding propagation code, but not the loop body.

**Direct evidence available:**

- `Network.recompute_all` is **declared** at `reasons/network.py:904`, with a docstring at line 907 that reads: *"Iterates to a fixpoint so cascading changes propagate regardless of..."* — the sentence is truncated. The only visible body line is `self._log("recompute", nid, new)` at `reasons/network.py:921`.
- A parallel implementation exists in the Postgres-backed store at `reasons/pg.py:1593`, whose docstring reads: *"Recompute truth values for all derived nodes to a fixpoint."* with a log call at `reasons/pg.py:1627`.
- The module docstring (`reasons/network.py:1-8`) states the design intent: it implements Doyle's (1979) TMS, truth values propagate automatically, retraction cascades, restoration cascades, and *"retracted nodes stay in the network (enables restoration without rederivation)."*
- Callers: `reasons/api.py:778, 1397, 1531, 1647`; `reasons/import_agent.py:317, 489` (both `len(network.recompute_all())` to count propagated changes); many tests.
- Tests pin the **invariants** the algorithm must satisfy:
  - `tests/test_network.py:147-149` — `recompute_all` must not resurrect a pinned/retracted node (`"recompute resurrected pinned node"`).
  - `tests/test_network.py:434-435` — same negation for `b`.
  - `tests/test_import_agent.py:172, 390-395` — OUT beliefs from an imported source stay OUT.
  - `tests/test_export_markdown.py:95,112,128` — calls `recompute_all()` before export snapshots.

### What the surrounding code lets us infer with confidence

**1. Multiple-justification semantics are kernel-level, not per-caller.** In `add_node` (`reasons/network.py:~110-147`), the initial truth value is chosen by:

```python
if node.justifications:
    node.truth_value = self._compute_truth(node)
else:
    node.truth_value = "IN"   # premise
```

So the answer to "how are multiple justifications combined" lives in `_compute_truth`, which is **not shown**. The module docstring mentions justification kinds are *"SL or CP"*, but the truth table for `(SL×n, CP×n)` combos is not visible. Multiple-justification behavior therefore cannot be confirmed from these observations.

**2. Ordering is handled by iteration, not by precedence.** The phrase *"Iterates to a fixpoint so cascading changes propagate regardless of..."* (line 907) is the key. It implies the algorithm does **not** rely on a single topological pass — it will repeatedly recompute derived nodes until nothing changes. That's the mechanism that makes order-dependence a non-issue: out-of-order updates in one pass are corrected in the next.

**3. Outlist changes are dependents too.** `_rebuild_dependents` (`reasons/network.py:~30-45`) registers `node.dependents` from both `j.antecedents` **and** `j.outlist`. So when the docstring says "regardless of" order, both the positive and negative (unless/outlist) sides of a justification trigger recomputation. This is confirmed by the derived-section logic in `reasons/derive.py` describing GATE / outlist semantics.

**4. Pins and `_retracted` are the resurrection guard.** `retract()` sets `node.metadata["_retracted"] = True` and `retracted_at`; the `assert_node()` path pops `_retracted`. The tests above pin the expectation that `recompute_all` consults this metadata (or an explicit pin flag) and does not flip such nodes back to IN when their *justifications* would technically support IN. Which exact flag is checked must be confirmed by reading the loop body — the observations only show `_retracted`, not a `_pinned` field.

**5. Parallel implementation is a hint at the loop shape.** `reasons/pg.py:1593` is a SQL-side equivalent ending with the same "fixpoint" wording. Comparing the Python fixpoint loop to the SQL version (which will likely use a recursive CTE or a `WHILE`-style repeated `UPDATE ... WHERE truth_value <> new_tv`) is a fast way to reconstruct the intended semantics.

**6. Callers treat the return as "the set of nodes whose TV changed."** `changed = net.recompute_all()` at `reasons/api.py:778/1397/1531`, and `len(network.recompute_all())` at `import_agent.py:317,489` — so the contract is a `list[str]` of changed IDs. Combined with the log call at line 921 (`self._log("recompute", nid, new)`), we can infer that inside the loop, whenever `new != old`, the code both appends to `changed` **and** logs.

### What is missing and must be read

These cannot be answered from the current observations:

1. **The exact loop in `recompute_all` (`reasons/network.py:904–~935`).** We only see the docstring at 904/907 and a log line at 921. We cannot tell whether it iterates over a topological order of `self.nodes`, a worklist seeded from changed nodes, or `deque`-based BFS (the module imports `deque` at line 13 — a strong hint at a worklist). We also can't see whether it processes the entire network each pass or only dependents of changed nodes.
2. **`_compute_truth` (`reasons/network.py`, not in the 0–200 slice).** This decides: for N justifications, is IN = "any justification satisfied" (classical Doyle), or a different combination? How are SL vs CP justifications combined? What happens with zero satisfied justifications on a derived node (out vs. unknown)? This is precisely the "multiple-justification behavior" the user wants confirmed.
3. **`_propagate` (`reasons/network.py`, used in `retract`/`assert_node`).** Whether `recompute_all` calls `_propagate` on each changed node or has its own loop. The symmetry between `retract` and `assert_node` (both call `_propagate`) suggests `_propagate` is a worklist confined to the reverse-dependency closure, whereas `recompute_all` is a full-network fixpoint.
4. **Pin flag name and check site.** `_retracted` is confirmed in `retract`; a pin mechanism is tested at `tests/test_network.py:147-149` but not shown in `network.py:0-200`. The guard site in `recompute_all` and in `_compute_truth` must be read.
5. **`reasons/pg.py:1593-1640`** — the SQL fixpoint loop, which will clarify the intended "update until no rows changed" semantics independent of Python dict iteration order.

In short: the observations **corroborate** that a fixpoint exists and that retracted/pinned nodes are protected, but they do **not** show the actual convergence loop or the multi-justification truth table.

## Topics to Explore

- [function] `reasons/network.py:recompute_all` — The truncated function at line 904; read the full loop to learn whether it uses a worklist `deque` (imported at line 13) or repeated full-network passes, and to confirm the pin/`_retracted` guard.
- [function] `reasons/network.py:_compute_truth` — The truth-function for a single node; this defines the multiple-justification semantics (SL/CP, any-vs-all) that the user asked about.
- [function] `reasons/network.py:_propagate` — The reverse-dependency cascade used by `retract`/`assert_node`; compare it to the `recompute_all` loop to see if they are the same algorithm or a localized vs. global variant.
- [function] `reasons/pg.py:recompute_all` — The Postgres analogue starting at line 1593; the SQL form usually expresses the fixpoint unambiguously (recursive CTE or `UPDATE ... WHERE tv <> new`), making the intended semantics clearer than Python dict-iteration order.
- [file] `tests/test_network.py` — Around lines 147-149 and 434-435, tests pin the resurrection guard; these are the fastest executable specification of the fixpoint invariants.

## Beliefs

- `recompute-all-is-defined-in-network-py` — `Network.recompute_all` is declared at `reasons/network.py:904` with return type `list[str]` and is invoked by callers in `reasons/api.py:778/1397/1531/1647` and `reasons/import_agent.py:317/489`.
- `recompute-all-iterates-to-fixpoint` — The docstring at `reasons/network.py:907` states the function "Iterates to a fixpoint so cascading changes propagate regardless of" traversal order, implying it is not a single topological pass.
- `recompute-all-exists-in-pg-backend` — A parallel implementation documented as "Recompute truth values for all derived nodes to a fixpoint" exists at `reasons/pg.py:1593`.
- `recompute-all-respects-retracted-metadata` — Tests at `tests/test_network.py:147-149` and `tests/test_import_agent.py:172` require `recompute_all` to leave nodes marked `_retracted` (or pinned) as OUT rather than resurrecting them.
- `outlist-nodes-are-dependents` — `Network._rebuild_dependents` (`reasons/network.py:~30-45`) registers target nodes as dependents of both `j.antecedents` and `j.outlist`, so outlist truth-state flips trigger recomputation as well as inlist flips.
