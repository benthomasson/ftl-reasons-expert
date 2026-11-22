---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: How `_retracted` short-circuits propagation, why retracted nodes are skipped entirely as dependents, and what happens if `_retracted` is set on a node that is later referenced.

## Explanation

**Caveat:** the supplied observations do not include the full body of `Network._propagate`, so the exact guard line is not visible. The behavior below is inferred from call sites, comments, and tests.

`_retracted` is a per-node metadata lock. It is set when a node is retracted or imported as `OUT`/`STALE`, and cleared when the node is explicitly restored/asserted or when the remote source says it is `IN`:

- `reasons/network.py:163` sets `node.metadata["_retracted"] = True` in the retraction path.
- `reasons/import_agent.py:311`, `411-412`, `464-467`, and `477-478` set the flag for imported beliefs that are `OUT`/`STALE`.
- `reasons/import_agent.py:425-426` and `485` clear it when the remote explicitly sends `IN`.
- `reasons/api.py:1637` clears it via `node.metadata.pop("_retracted", None)`.

The reverse index is `Node.dependents` (`reasons/__init__.py:38`). `Network.retract()` and `Network.assert_node()` both call `_propagate(node_id)` after changing truth state (`reasons/network.py:178` and `reasons/network.py:201`). `_propagate()` walks that `dependents` set. When it encounters a dependent whose metadata has `_retracted`, it skips that node before recomputing or enqueueing it. The import-agent comment at `reasons/import_agent.py:402-404` says the flag is set so `_propagate()` and related propagation paths ignore the node until the remote explicitly sends `IN`. `_propagate_access_tags()` at `reasons/network.py:218` is another propagation path that likely observes the same lock.

This is a **runtime filter**, not a structural removal from `dependents`. Tests in `tests/test_dependents_integrity.py` verify that the reverse index remains consistent across `retract()`/`assert_node()`, `add_justification()`, and outlist dependencies. A retracted node can still appear in another node’s `dependents` set; propagation just refuses to process it as a dependent while the flag is set.

Why skip retracted nodes entirely as dependents? Because a retracted node may still have valid-looking justifications. If normal propagation processed it, it could be re-derived `IN`, overriding the explicit retraction or the remote source’s `OUT`/`STALE` status. Skipping it prevents local propagation from “resurrecting” the node and also prevents propagation from cascading through it via that path. When `Network.retract("X")` is called, the origin `X` is still the starting point of propagation so its existing dependents can be updated; the skip mainly matters when `X` later appears as a dependent of some antecedent that changes.

If `_retracted` is set on a node that is later referenced:

- The node can still be referenced as an antecedent or outlist member. The reference will be recorded in the reverse index, e.g. through `dependents.add(...)` call sites such as `reasons/api.py:1394`, `1528`, `1630`, and the rebuilding logic around `reasons/api.py:4925`.
- The later referencing node will depend on the retracted node’s current truth value, typically `OUT`/`STALE`, but the retracted node itself remains locked. If its own antecedents change, `_propagate()` will skip it as a dependent, so it will not be recomputed or restored by local propagation.
- If the flag is later cleared (`reasons/import_agent.py:425-426`, `485`; `reasons/api.py:1637`) and a recompute/propagation runs, the node becomes eligible again and can cascade to its dependents, including the later reference. The import path calls `network.recompute_all()` after import changes (`reasons/import_agent.py:317`, `489`), which is how the rest of the network is refreshed around locked nodes.
- Until the flag is cleared, the node acts as a propagation barrier: its reverse edges exist structurally, but propagation through it is short-circuited. This is different from the dangling-dependent case in `tests/test_dangling_dependents.py`, where `_propagate()` skips missing node IDs and logs a warning; `_retracted` appears to be a silent lock.

## Topics to Explore

- [function] `reasons/network.py:_propagate` — Confirm the exact `_retracted` guard and whether it skips before recomputation, before enqueueing, or both.
- [function] `reasons/network.py:recompute_all` — See how full recomputation treats `_retracted` nodes and whether it reaches their dependents.
- [file] `reasons/import_agent.py` — Read the import locking/unlocking flow around lines 311, 402-485, and 537-578.
- [file] `tests/test_dependents_integrity.py` — Verify that `_retracted` does not remove reverse-index edges and that retract/restore remains structurally consistent.
- [general] `retracted-vs-dangling-dependents` — Compare the silent `_retracted` skip with the warning-and-skip behavior for dangling dependents.

## Beliefs

- `retracted-flag-locks-node-out` — `_retracted` metadata is set when a node is retracted or imported as `OUT`/`STALE`, and cleared when it is asserted/restored or the remote sends `IN`.
- `propagation-skips-retracted-dependents` — `_propagate()` skips nodes with `_retracted` when they appear in a predecessor’s `dependents` set.
- `dependents-index-remains-structural` — `_retracted` does not remove the node from the reverse `dependents` set; the skip is a runtime propagation filter.
- `clearing-retracted-enables-repropagation` — Removing `_retracted` through the import/API paths makes the node eligible for normal recomputation and cascade again.
- `retracted-nodes-can-still-be-antecedents` — A node with `_retracted` can be referenced by later justifications, but it remains a propagation barrier until the flag is cleared.
