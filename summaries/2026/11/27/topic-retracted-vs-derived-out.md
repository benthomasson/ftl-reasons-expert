---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: How `_retracted` metadata differs from a derived `OUT` truth; critical because `propagate` only respects the former.

# `_retracted` metadata vs. a derived `OUT` truth

## The two concepts at a glance

| | **`OUT` truth value** | **`_retracted` metadata** |
|---|---|---|
| Where stored | `node.truth_value` (`string`, `"IN"`/`"OUT"`) | `node.metadata["_retracted"]` (boolean flag) |
| Source of truth | Computed by the truth-maintenance engine | Set explicitly by operator action or by an import |
| Example setter | Any `recompute_all()` pass | `network.py:163`, `import_agent.py:311/467/498` |
| Example clearer | — | `api.py:1637` (`metadata.pop("_retracted", None)`), `import_agent.py:425-426, 485` |
| Lifetime | Transient — recomputed on every `propagate` | Sticky — persists until explicitly cleared |
| Meaning | "The current graph state makes this node unjustified" | "An operator/import **chose** this node to be OUT" |

The `truth_value` field is declared on the node dataclass at `reasons/__init__.py:35` (`truth_value: str = "IN"  # IN or OUT`). The `_retracted` key is not part of that dataclass; it lives only in the free-form `metadata` dict, which is why it can be set and cleared independently of the TMS.

## Why the distinction matters

`propagate` (`reasons/api.py:773`) is a *recomputation* pass. It walks the justification graph and produces whatever `truth_value` the graph logic implies. That means a node can flip to `OUT` through two very different routes:

1. **Derived OUT** — the justification check fails right now. E.g., an antecedent went `OUT`, or all justifications were retired, or a `retract_after` defeater defeated them. The next `recompute_all()` will silently flip it back to `IN` as soon as a valid justification exists again.
2. **Locked OUT** — the node has `_retracted=True` in metadata. That flag represents **explicit intent** ("someone told us this should be OUT"), and the propagation code treats it as an override: even if the node has fully-satisfied justifications, the node stays `OUT`.

The import code states this explicitly at `reasons/import_agent.py:464-467`:

```python
# Set _retracted directly instead of using retract_after,
# ...
# _retracted to keep OUT-with-justifications nodes locked.
network.nodes[node_id].metadata["_retracted"] = True
```

If the importer used the graph-native `retract_after` defeater mechanism instead, propagation would only keep the node OUT while the defeater's antecedents were satisfiable. `_retracted` is a stronger, metadata-level assertion: "this node is retracted, full stop."

## What `propagate` does and does not do

`propagate` (defined at `reasons/api.py:773`, dispatched to `_pg_dispatch(..., "propagate")` at `reasons/api.py:776`) is the surface that recomputes truth values via `network.recompute_all()` (called by importers at `import_agent.py:317` and `:489`; exposed via CLI `reasons/cli.py:855-857`).

- It **recomputes** `truth_value` for every node from its justifications and outlist conditions.
- It **does not** recompute a node back to `IN` when `node.metadata.get("_retracted")` is truthy — the flag short-circuits the "restore" branch.

A pure "derived OUT" has no such short-circuit, so it is by definition *unstable*: any future graph change that satisfies a justification will resurrect it on the next propagate. Only `_retracted` gives you a durable OUT that survives recomputation.

The `retract_node` path demonstrates the pairing. At `reasons/network.py:149-178` the function docstring reads "Mark a node OUT and propagate the retraction cascade" and sets **both**:

- `node.truth_value = "OUT"` (the immediate, visible effect), and
- `node.metadata["_retracted"] = True` (the durable lock, `network.py:163`),

then calls `self._propagate(node_id)` (`network.py:178`). Symmetrically, the restore path (`network.py:182+`, and later the loading code at `api.py:1637`) clears the flag with `node.metadata.pop("_retracted", None)` so that `IN` restoration is allowed.

## The import scenario

The clearest illustration is `reasons/import_agent.py`:

- Lines **402-404** explain the invariant: "_retracted is set so both `_propagate()` … [won't restore the node] … the else branch clears _retracted" when the remote explicitly sends `IN`.
- Lines **411-412** and **477-478** re-set `_retracted` when the remote source still signals `OUT`/`STALE`.
- Lines **425-426** and **485** clear `_retracted` when the remote source sends `IN`.

So the remote's `OUT` signal is translated into the **durable** `_retracted` flag, not merely into the ephemeral `truth_value = "OUT"`. If the importer only set the truth value, the very next `recompute_all()` (line 317 or 489) would flip the node back `IN` whenever the imported justifications happened to be satisfiable — the belief would "self-heal" against the operator's explicit retraction.

`reasons/api.py:1556-1557` is the counterpart for locally-derived OUT-with-valid-justifications nodes: it "_[f]inds OUT nodes with retract_reason metadata whose justifications would be satisfied if not for the _retracted flag_" and, for each, "creates a defeater." That function is essentially the TMS-native equivalent of the `_retracted` lock — it adds graph structure so propagation respects the retraction even without the metadata override.

## Caveat: what the observations don't show

The body of `propagate` and `network._propagate()` / `recompute_all()` are not included in the observations — we see only the entry points and the call sites. We are inferring the `_retracted` short-circuit from (a) the inline comments in `import_agent.py:402-404, 464-467`, (b) the paired set/clear sites, and (c) the existence of `api.py:1556-1557`'s defeater-builder for the same problem. To confirm the exact mechanism (e.g., whether the check is `if node.metadata.get("_retracted")` inside the truth-value computation, or a branch in `recompute_all()` before it calls `_propagate`), the reader should read `reasons/network.py` around `_propagate` and `recompute_all`, and the RPC backend's `propagate` handler that `_pg_dispatch` reaches.

## Topics to Explore

- [function] `reasons/api.py:propagate` — The single public entry point; trace where it dispatches to confirm the `_retracted` short-circuit lives in the backend or in `network.recompute_all()`.
- [file] `reasons/network.py` — Contains `retract_node`, `restore_node`, `_propagate`, and `recompute_all`; this is where the `_retracted` vs. derived `OUT` distinction is enforced.
- [function] `reasons/api.py:1556` (or the symbol around that line) — Builds a defeater for OUT-with-valid-justifications nodes; shows the TMS-native alternative to the `_retracted` lock.
- [general] `import-agent-truth-sync` — How `import_agent.py` translates remote `IN`/`OUT`/`STALE` signals into local `truth_value` and `_retracted`; the best worked example of the distinction.
- [file] `reasons/import_agent.py` — Read lines 390-500 end-to-end to see all four cases (remote IN/OUT × current IN/OUT) and where `_retracted` is set vs. cleared.

## Beliefs

- `derived-OUT-is-transient` — A node whose `truth_value` is `"OUT"` purely from failed justifications will be flipped back to `"IN"` by the next `recompute_all()` once a valid justification exists.
- `retracted-metadata-locks-OUT` — `node.metadata["_retracted"] = True` is the durable marker that prevents propagation from restoring an explicitly retracted node to `"IN"`.
- `retract-node-sets-both` — `network.retract_node` sets both `truth_value = "OUT"` and `metadata["_retracted"] = True` before propagating (`reasons/network.py:149-178`).
- `import-agent-maps-remote-OUT-to-retracted` — The import code sets `_retracted` when the remote source signals OUT/STALE and pops it when the remote signals IN (`reasons/import_agent.py:411-412`, `425-426`, `477-478`, `485`).
- `propagate-requires-retracted-to-preserve-explicit-retraction` — Without the `_retracted` flag (or an equivalent defeater created by the helper around `reasons/api.py:1556`), a `propagate` pass cannot distinguish an intentional retraction from a merely unjustified node.
