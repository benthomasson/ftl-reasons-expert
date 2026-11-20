---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Investigate how `sl` dependents are discovered and forced to `OUT` after source retraction.

# How `sl` Dependents Are Discovered and Forced to `OUT`

## Caveat: the observations are incomplete

The supplied grep results are internally contradictory: `grep_dependents_usage`, `grep_retraction_logic`, and `grep_out_status_usage` all report **0 matches**, yet the included test file `tests/test_dangling_dependents.py` repeatedly calls `net.retract(...)`, reads/writes `net.nodes[...].dependents`, and asserts `truth_value == "OUT"`. `find_dependents_symbols` also reports `count: 0`.

That means the searches were almost certainly mis-scoped (e.g. limited to the `reasons_lib/` re-export shims, or a bad regex), and the actual implementation files were not read. The explanation below is therefore reconstructed from the **API surface** (`reasons/api.py`, `reasons/cli.py`, `reasons/derive.py`) plus the **behavioral contract** encoded in `tests/test_dangling_dependents.py`. Where the real code should live is flagged explicitly.

---

## 1. How an SL dependent is registered

An SL justification is just a list of antecedent node IDs:

- CLI: `--sl A,B` argument is declared in `reasons/cli.py:3076`, `reasons/cli.py:3094`, and `reasons/cli.py:3376`.
- API parsing: `reasons/api.py:280–323` (`sl: str = ""` at line 280, then split at line 323: `[a.strip() for a in sl.split(",")]`) and the duplicate path at `reasons/api.py:408–440`.
- Derivation: `reasons/derive.py:655` builds an SL string from `p["antecedents"]` and passes it straight into justification creation at `reasons/derive.py:660`.

The tests show the resulting graph shape:

```python
Justification(type="SL", antecedents=["a"])
```

and, once node `b` exists with that justification, `net.retract("a")` causes `b` to become `OUT`. Therefore adding an SL justification must register `b` in `a.dependents` — the reverse edge. In the tests dependents are manipulated as a **plain `set` of node IDs** on the node object:

```python
net.nodes["a"].dependents.add("nonexistent")
net.nodes["a"].dependents.update({"ghost1", "ghost2", "ghost3"})
```

The registration code itself is **not in the observations**. It should be in the node/network layer — most likely `reasons/node.py` (where `dependents` is a `set`) and the `add_justification`/`add_node` paths in `reasons/network.py`.

---

## 2. How dependents are discovered during retraction

`Network.retract(node_id)` is what the tests call. Based on the test hints, it delegates to a shared propagation routine (variously described as “`assert_node` also triggers `_propagate`”, “`challenge()` path triggers `_propagate`”, “`add_justification()` can trigger `_propagate`” in `tests/test_dangling_dependents.py`).

That `_propagate` routine:

1. Starts from the retracted node and walks its `dependents` set.
2. For each dependent ID, looks it up in `net.nodes`.
3. **Dangling handling** — if the ID is absent:
   - emits a log entry `{"action": "warn", "target": <id>, "value": "dangling ... <parent-id>", "timestamp": ...}` — verified by `test_single_warning_logged`, `test_warning_includes_parent_node_id`, `test_warning_has_timestamp`, `test_multiple_dangling_produce_multiple_warnings`;
   - does **not** add the ID to `visited` (`test_dangling_does_not_enter_visited`);
   - does **not** add the ID to the returned `changed` list (`test_dangling_not_in_changed_list`);
   - continues processing other dependents (`test_valid_dependent_still_retracted`, `test_chain_with_dangling_in_middle`, `test_diamond_with_dangling`).
4. Because the routine is shared, the same warning behavior fires from `assert_node`, `challenge`, and `add_justification` (`test_warning_on_assert_path`, `test_challenge_with_dangling`, `test_add_justification_with_dangling`).

The graph-walk semantics are:

- `retract("a")` → walk `a.dependents` → for each valid dependent, re-evaluate its justifications → if no justification remains satisfied, force it `OUT` → recurse into its dependents.
- Tests confirm the recursion handles chains (`A → B → C` in `test_chain_with_dangling_in_middle`) and diamonds (`A → B, A → C, B&C → D` in `test_diamond_with_dangling`).

The actual iteration order/queue-vs-stack is **not observable** here; it lives in the propagation implementation that was not read.

---

## 3. How the dependent is forced to `OUT`

Each `Node` exposes a `truth_value` attribute. Tests assert:

```python
assert net.nodes["b"].truth_value == "OUT"
```

after the source is retracted, and

```python
assert net.nodes["b"].truth_value == "IN"
```

after `assert_node("a")` (see `test_valid_dependent_still_restored`, `test_valid_dependent_still_retracted`).

The forcing logic is: an SL justification is “supported” iff all of its antecedents are `IN`. After retraction of a source, the dependent has zero supported justifications, so `_propagate` assigns `truth_value = "OUT"` and enqueues its own dependents. `Network.retract` returns the list of IDs whose status changed; the tests rely on that list containing `"b"`/`"c"` but **not** `"ghost"` (`test_valid_dependent_still_retracted`, `test_dangling_not_in_changed_list`).

The concrete assignment line (the one that writes `"OUT"` onto the node) is **not present in the observations** — `grep_out_status_usage` returned 0 matches, which again points to a scoping problem, since the tests unambiguously exercise that string literal.

---

## 4. Where to look next in the real code

The re-export shims tell us where the real modules are:

- `reasons_lib/storage.py` → `from reasons.storage import *`
- `reasons_lib/derive.py` → `from reasons.derive import *`

So the `Network.retract` / `_propagate` implementation almost certainly lives in `reasons/network.py`, with `Node.dependents` and `Node.truth_value` defined in `reasons/node.py` (both files were not included in the observations). Reading those two files is required to move from “behavioral contract inferred from tests” to “verified implementation.”

---

## Topics to Explore

- [file] `reasons/network.py` — Contains `Network.retract`, `assert_node`, `challenge`, `add_justification`, and the shared `_propagate` traversal where dependents are discovered and forced `OUT`; this is the missing implementation behind every test in `tests/test_dangling_dependents.py`.
- [file] `reasons/node.py` — Defines the `Node.dependents` set (IDs of nodes whose SL justifications reference this node) and the `truth_value`/`"OUT"` state that propagation mutates.
- [function] `reasons/network.py:_propagate` — The shared BFS/DFS over `dependents` that re-evaluates SL justifications, skips dangling IDs with a `warn` log, avoids adding dangling IDs to `visited`, and excludes them from `changed`.
- [function] `reasons/api.py:add_justification` — Entry point around `reasons/api.py:280–323` / `408–440` that parses the comma-separated `sl` string and (indirectly) registers reverse `dependents` edges on each antecedent.
- [general] `dangling-dependents-log-schema` — The `{"action": "warn", "target", "value", "timestamp"}` log contract used for observability of skipped dependents; worth verifying against `net.log` consumers.

---

## Beliefs

- `sl-antecedents-parsed-from-comma-separated-string` — SL antecedents arrive as a comma-separated `sl` string and are split at `reasons/api.py:323` and `reasons/api.py:440`.
- `node-dependents-is-set-of-node-ids` — Each `Node.dependents` is a `set` of node IDs (evidenced by `.add(...)` and `.update({...})` usage throughout `tests/test_dangling_dependents.py`).
- `retract-forces-unsupported-sl-dependents-out` — `Network.retract` cascades through `dependents` and sets `truth_value = "OUT"` on nodes whose SL justifications no longer have all antecedents `IN`.
- `propagate-emits-warn-for-dangling-dependents` — Missing dependent IDs produce `{"action": "warn", "target": <id>, "value": "dangling ...", "timestamp": ...}` log entries instead of raising `KeyError`.
- `propagate-excludes-dangling-from-visited-and-changed` — Dangling IDs are skipped, never added to the `visited` set, and never appear in the `changed` list returned by `retract`.
- `reasons-lib-modules-re-export-from-reasons` — `reasons_lib/storage.py` and `reasons_lib/derive.py` are 3-line `from reasons.* import *` shims, so implementations live under the `reasons/` package.
