---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: How `dependents` sets are populated incrementally by `add_node` / `add_justification` and re-hydrated in `export_network` / `import_json`.

# How `dependents` Sets Are Populated Incrementally and Re-Hydrated

## The shape of the index

Every `Node` carries a `dependents: set[str]` — a **reverse index** mapping a node ID to the IDs of nodes whose justifications reference it (either as an `antecedent` in the inlist, or as an entry in the `outlist`). The reverse index is *derived state*: it is functionally `{dep_id for dep in nodes for j in dep.justifications for ref in j.antecedents + j.outlist}`.

Because it's derived, the invariant is that it must match what `verify_dependents()` recomputes on the fly (`reasons/network.py`):

```python
def verify_dependents(self) -> list[str]:
    expected = {nid: set() for nid in self.nodes}
    for node in self.nodes.values():
        for j in node.justifications:
            for ant_id in j.antecedents:
                if ant_id in self.nodes:
                    expected[ant_id].add(node.id)
            for out_id in j.outlist:
                if out_id in self.nodes:
                    expected[out_id].add(node.id)
    # compare live vs. expected, return error strings
```

Note that `verify_dependents()` is **read-only** — it does not repair the index (confirmed by `tests/test_dependents_integrity.py::TestVerifyDependents::test_does_not_modify_state`).

---

## 1. Incremental population — `Network.add_node`

In `reasons/network.py`, `Network.add_node` builds the new node and then, *before* registering it in `self.nodes`, mirrors every outbound justification edge into the dependent sets of the referenced nodes:

```python
node = Node(id=id, text=text, justifications=justifications or [], ...)

# Register as dependent of antecedents (inlist) and outlist nodes.
# Both can affect this node's truth value when they change.
for j in node.justifications:
    for ant_id in j.antecedents:
        if ant_id in self.nodes:
            self.nodes[ant_id].dependents.add(id)
    for out_id in j.outlist:
        if out_id in self.nodes:
            self.nodes[out_id].dependents.add(id)

self.nodes[id] = node
```

Two consequences worth noting:

1. **Forward references are silently skipped.** The guard `if ant_id in self.nodes` means that if you `add_node("b", sl=["a"])` before `add_node("a")`, `a.dependents` never learns about `b`, and nothing in `add_node` retroactively scans existing nodes to find references to the newly-inserted `a`. The index is correct only for nodes that already existed at insert time.
2. **The new node's own `dependents` set starts empty** (the `Node` default). It will only be populated later when some other node derives from it, or by `_rebuild_dependents()`.

The public `reasons/api.py:277` `add_node` is the string-facing wrapper (`sl="a"`, `unless="b"` → constructs a `Justification`); the edge bookkeeping lives exclusively in `Network.add_node`. The grep for the literal `dependents` in `reasons/api.py` returns zero hits — the API layer never touches the index directly.

---

## 2. Incremental population — `Network.add_justification`

`Network.add_justification` is defined at `reasons/network.py` (body truncated in the observation at line 500) and is called from `reasons/api.py:406` and `reasons/mcp_server.py:204` (the MCP tool wrapper). The test `tests/test_dependents_integrity.py::TestMutationPathIntegrity::test_after_add_justification` pins down the contract:

```python
net.add_justification("c", Justification(type="SL", antecedents=["b"]))
assert "c" in net.nodes["b"].dependents     # reverse edge was mirrored
assert_clean(net)
```

So *by design*, `add_justification` does the same mirroring loop as `add_node` for the new justification, then recomputes truth value and propagates through the freshly-grown `dependents` set. (The exact body is **missing** from the observations; only the signature line `def add_justification(self, node_id: str, justification: "Justification") -> dict:` is visible. The test is the primary evidence that it mirrors both `antecedents` and `outlist`.)

Other mutation paths in `Network` maintain the same invariant, as asserted by `TestMutationPathIntegrity`:

- `supersede()` — adds an `outlist` entry and directly calls `self.nodes[new_id].dependents.add(target_id)` (`test_after_supersede` expects `"old" in net.nodes["new"].dependents`).
- `challenge()` / `defend()` — `test_after_challenge` / `test_after_challenge_and_defend`.
- `summarize()` — `test_after_summarize`.
- `convert_to_premise()` — removes justifications and *removes* the reverse edge (`test_after_convert_to_premise` asserts `"b" not in net.nodes["a"].dependents`).

---

## 3. Canonical repair — `Network._rebuild_dependents`

Because incremental population depends on insertion order and on every mutation path being disciplined, the codebase has a single canonical repair:

```python
def _rebuild_dependents(self) -> None:
    for node in self.nodes.values():
        node.dependents = set()
    for node in self.nodes.values():
        for j in node.justifications:
            for ant_id in j.antecedents:
                if ant_id in self.nodes:
                    self.nodes[ant_id].dependents.add(node.id)
            for out_id in j.outlist:
                if out_id in self.nodes:
                    self.nodes[out_id].dependents.add(node.id)
```

The docstring is explicit: *"This is the single canonical implementation. All other rebuild call sites should delegate here."* Tests confirm it clears stale entries, is idempotent, and repairs both over- and under-populated sets (`tests/test_dependents_integrity.py::TestRebuildDependents`, `TestCorruptionAndRepair`).

---

## 4. Re-hydration across `export_network` / `import_json`

### `export_network` — `reasons/api.py:1814`

The body is **not in the observations**, but the `dependents` grep in the API layer returned 0 hits, and the round-trip tests show that what is persisted is the *justifications* themselves (`antecedents`, `outlist`, `type`, `label`, hashes), not the reverse index. That is the natural design: `dependents` is purely derived, so serializing it would be redundant and would double the risk of drift. The `round_trip` tests are the contract:

- `tests/test_import_json.py::test_round_trip_justifications` — `b`'s justifications come back with `antecedents == ["a"]` and `truth_value == "IN"`.
- `tests/test_import_json.py::test_round_trip_outlist` — `x`'s justification comes back with `outlist == ["y"]`, and **outlist propagation still works after import** (`api.assert_node("y")` flips `x` to `OUT`). This is the key proof that whatever path populated `dependents` after import left `y.dependents ⊇ {"x"}` intact.

### `import_json` — `reasons/api.py:2104`

Body **not shown**. Two plausible mechanisms are compatible with the tests:

1. **Insertion-order rebuild.** Iterate the exported `nodes` array and call `Network.add_node` (or the API wrapper) for each node. If the export preserves a topological order (premises before derived), `add_node`'s incremental mirroring yields a correct index. But `test_round_trip_outlist` shows `x` (derived, with `outlist=["y"]`) exported *after* `y` was retracted, so ordering is not guaranteed to be topological in the usual sense.
2. **Explicit `_rebuild_dependents()` after all nodes are inserted.** This is the robust mechanism and the one the code seems designed for.

The `Storage` round-trip test is the tie-breaker:

```python
# tests/test_dependents_integrity.py::TestStorageRoundTripDependents
assert_clean(loaded)
assert "c" in loaded.nodes["a"].dependents
assert "d" in loaded.nodes["b"].dependents
```

`assert_clean` calls `verify_dependents()`. For that to pass after a save/load cycle involving an outlist node (`d` with `outlist=["b"]`) and a diamond (`a → b,c; b,c → d`), some explicit rebuild must run in the load/import path — `add_node`'s incremental mirroring alone cannot guarantee correctness if camera order is not topological. So the intended contract is:

> After `export_network` → `import_json` (or `Storage.save` → `Storage.load`), `dependents` is guaranteed consistent because the load path constructs nodes and then rebuilds the index via `_rebuild_dependents()` (or equivalent).

**What is missing to fully confirm this:** the bodies of `reasons/api.py:1814` (`export_network`), `reasons/api.py:2104` (`import_json`), `reasons/api.py:406` (API `add_justification` wrapper), `reasons/storage.py` (`save`/`load`), and the tail of `Network.add_justification` (lines ~500+ of `reasons/network.py`). With those we could identify the exact line where the rebuild happens and confirm whether `import_json` delegates to `add_node` or does a bulk insert plus rebuild.

---

## 5. Summary of the lifecycle

| Stage | Who touches `dependents` | Mechanism |
|---|---|---|
| Node created | `Network.add_node` | Mirror `antecedents` + `outlist` into existing targets' sets |
| Justification added | `Network.add_justification` | Same mirror loop for the new justification |
| Supersede / challenge / defend / summarize | Respective `Network` methods | Direct `dependents.add(...)` alongside justification mutation |
| Convert-to-premise | `convert_to_premise` | Removes the reverse edge alongside justification removal |
| De-novo reverse index | `_rebuild_dependents` | Clear + replay all justifications — canonical repair |
| Consistency check | `verify_dependents` | Read-only recompute; returns error strings |
| Export | `export_network` (api.py) | Serializes justifications only (no `dependents` in JSON) |
| Import | `import_json` (api.py) | Re-creates nodes + justifications, rebuilds the index (statement of intent; exact call site not shown) |

---

## Topics to Explore

- [function] `reasons/api.py:export_network` — Confirm whether `dependents` is ever serialized and how justifications are shaped in the JSON payload.
- [function] `reasons/api.py:import_json` — Find the exact place where `_rebuild_dependents()` (or equivalent) is invoked after all nodes are inserted.
- [file] `reasons/storage.py` — Verify whether `Storage.save`/`Storage.load` also rely on the reverse index rebuild and whether the SQLite schema stores justifications or edges.
- [function] `reasons/network.py:add_justification` — Read the tail of the function to confirm that it mirrors both `antecedents` and `outlist`, and check whether it ever calls `_propagate_access_tags` or modifies dependents of new targets that were previously unreferenced.
- [general] `forward-reference-edge-cases` — Probe whether `add_node("b", sl=["a"])` before `add_node("a")` leaves a stale index, and whether `import_json` is the only consumer that depends on `_rebuild_dependents()` to fix it.

## Beliefs

- `dependents-is-derived-not-persisted` — The `dependents` set on each `Node` is recomputed from justifications and is not serialized by `export_network`; no `dependents` references exist in `reasons/api.py`.
- `add-node-mirrors-justification-edges` — `Network.add_node` mirrors every `antecedents` and `outlist` entry of the node's justifications into the target node's `dependents` set before the new node is registered in `self.nodes`.
- `add-node-skips-forward-references` — At `add_node` time, edges whose target is not yet in `self.nodes` are silently dropped, so insert order can leave the reverse index stale until `_rebuild_dependents()` runs.
- `rebuild-dependents-is-canonical-repair` — `Network._rebuild_dependents` (declared "the single canonical implementation") clears all dependents and replays every justification; `verify_dependents` is a read-only equivalent used for detection, not repair.
- `supersede-updates-dependents-directly` — `Network.supersede` explicitly calls `self.nodes[new_id].dependents.add(target_id)` when wiring the new node's `outlist`, which is what makes `"old" in net.nodes["new"].dependents` hold in the supersede test.
- `add-justification-mirrors-new-edges` — `Network.add_justification` adds the host node to the `dependents` set of every new antecedent/outlist target, as asserted by `TestMutationPathIntegrity::test_after_add_justification`.
