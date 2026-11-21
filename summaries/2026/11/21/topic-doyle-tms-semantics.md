---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Doyle's original TMS distinguishes "in/out" from "retracted/pinned"; understanding which tests encode TMS proper vs. this project's sticky extensions will clarify `_retracted` design.

# Doyle's TMS vs. this project's sticky extensions

## Doyle's original: IN/OUT is the truth value

The `reasons/network.py` module docstring (lines 1–9) is explicit about the base model:

- Nodes have justifications (`SL` with an inlist + outlist, or `CP`).
- Truth values propagate automatically — `IN` means currently believed, `OUT` means not believed.
- Retraction cascades when a node goes `OUT`; restoration cascades when it comes back `IN`.
- Retracted nodes stay in the network so restoration doesn't require rederivation.

In Doyle's framing there is exactly one truth state, `truth_value ∈ {IN, OUT}`, and it is **derived, not asserted**: `add_node` computes it at insertion time (`network.py` around line 133–135: "If no justifications are provided, the node is a premise (IN by default). If justifications are provided, truth value is computed from them"), and `_propagate` recomputes dependents whenever an antecedent or outlist node changes value.

`tests/test_outlist.py` is the clearest example of **pure TMS** in the repo:
- `test_outlist_valid_when_out` / `test_outlist_invalid_when_in` prove inlist/outlist truth-table semantics.
- `test_assert_outlist_node_retracts_dependent` and `test_retract_outlist_node_restores_dependent` prove the *cascade* and *restoration* rules.
- `test_alternate_justification_survives_outlist` proves OR-justification semantics.
- `test_outlist_node_not_in_network` proves "absent = OUT" (an ID not in the network is not believed).

Note what these tests **never** assert: no `_retracted` reads, no metadata pops, no pinned SHA. Only `node.truth_value`. That is the boundary of "TMS proper."

`tests/test_backtracking.py` is *mostly* TMS proper as well — `trace_assumptions`, `add_nogood`, `find_culprits`, and the "retract a premise, not a derived node" behaviour are Doyle-style dependency-directed backtracking. The **`TestEntrenchment`** class is the one exception: Doyle's 1979 paper does not score entrenchment; that heuristic is a project invention for picking which culprit to retire (`network.py:_entrenchment`, ~lines 297–337, adds scores for premise status, source presence, dependent count, and `beliefs_type` metadata).

## `_retracted` is not `truth_value` — it is a sticky side-channel

The `_retracted` field is a **metadata flag** (`node.metadata["_retracted"]`), not a truth value. The codebase uses it two different ways, and both are extensions over Doyle:

### 1. A "manual retraction" annotation in the core network

In `reasons/network.py`:

- Line **163**, inside `retract()`, does `node.metadata["_retracted"] = True` alongside `node.truth_value = "OUT"` and a `retracted_at` timestamp.
- Line **184**, inside `assert_node()`, does `node.metadata.pop("_retracted", None)` alongside restoring `truth_value = "IN"`.

Note the early-exit at the top of `retract()` (visible in the read of `network.py` around line 155–160):

```python
if node.truth_value == "OUT":
    node.metadata["_retracted"] = True
    node.retracted_at = now
    ...
    return []
```

So `_retracted` is set **even if the node is already `OUT`** — a deliberate behaviour called out in `CHANGELOG.md:34` ("`_retracted` flag now set even when node is already OUT"). This is the stepping stone to the second, more important use.

### 2. A "locked" state for imported / remote "OUT/STALE" nodes

`reasons/import_agent.py` is where `_retracted` acquires semantics well beyond Doyle. Key lines:

- **311**, **411–412**, **477–478**, **498–499**: every path that receives "OUT" or "STALE" from the remote source sets `_retracted = True`.
- **425–426**, **485**: the "remote explicitly sends IN" path pops `_retracted`.
- **464–467**: the telling comment —
  ```python
  # Set _retracted directly instead of using retract_after,
  # ... _retracted to keep OUT-with-justifications nodes locked.
  network.nodes[node_id].metadata["_retracted"] = True
  ```
- **402–404**: "remote flips it to IN. `_retracted` is set so both `_propagate()` …" — i.e. `_propagate` is expected to honour `_retracted` so that a node whose justifications would compute back to `IN` stays locked `OUT` until the source explicitly says `IN`.
- **537** and **578** summarise the contract: "Beliefs that are OUT/STALE in the source are marked with `_retracted`" and "`_retracted` flag cleared when remote says IN".

This is **not** in Doyle. Doyle's `OUT` is a *derived* state that flips back to `IN` automatically when justifications become satisfiable. The project needs a *sticky* `OUT` — a node that was marked `OUT` externally and must not be naively resurrected by local propagation. That's what `_retracted` buys.

`reasons/cli.py:911–912` and `:942–943` make the distinction visible at the UX layer:

```
Retracted: N (STALE/OUT in source)
Retracted: N (OUT/STALE in source)
```

The CLI is deliberately surfacing this as a **different** category from the ordinary `OUT` set. Alternatively, `reasons/api.py:1557` describes an API function that inspects `_retracted` and "creates a defeater" for each such node — i.e. the API turns a sticky-retracted node into an outlist-protected `OUT` so ordinary propagation can't undo it.

## Which tests encode which concept

| Test file | Category | Why |
|---|---|---|
| `tests/test_outlist.py` | **TMS proper** | Only `truth_value`, inlist/outlist, cascade, restoration, persistence. Zero references to `_retracted`. |
| `tests/test_backtracking.py` (classes `TestTraceAssumptions`, `TestFindCulprits`, `TestBacktrackingInNogood`, `TestBacktrackingAPI`) | **TMS proper** | Dependency-directed backtracking via `add_nogood`. `test_inactive_nogood_no_backtracking` uses `net.retract("b")` purely to set `truth_value = "OUT"`. |
| `tests/test_backtracking.py::TestEntrenchment` | **TMS proper + sticky heuristic** | Exercises `_entrenchment` scoring (`network.py:_entrenchment`), which Doyle does not specify. It is what lets `test_nogood_retracts_speculation_not_evidence` protect sourced premises from unrelated speculation. |
| `tests/test_hide_out.py` | **UX extension** | Exercises a *display/filter* extension at the API layer (`api.search`/`lookup`/`compact` with `include_out=True`). It uses `api.retract_node` to produce `OUT`, but does not test `_retracted` stealing — it just tests whether `OUT` nodes are hidden by default. |
| `tests/test_pin_sources.py` | **Sticky extension, but a different one** | Tests pinning a git `SHA` to a node's source (`pinned_sha` metadata) and checking staleness against later commits. `test_skips_out_nodes` uses `truth_value == "OUT"` only as a skip filter — pinning works on `IN` nodes. Completely orthogonal to `_retracted`. |

Two important observations from this mapping:

1. **`_retracted` is tested almost nowhere in the diff we can see.** The grep pattern over test files (`retracted|pinned|in/out|outlist|inlist|hide_out`) returned **zero** matches, which is suspicious given how much `_retracted` is used in `import_agent.py`. Either the direct `_retracted` tests live in a file that wasn't scanned, or the tests are effectively integration-only (via `import_agent` round-trips).
2. **`tests/test_hide_out.py` looks "sticky" but is not.** It is purely a presentation filter over `truth_value`; it does not touch `_retracted` and would pass on a vanilla Doyle network fed through the API.

## What this clarifies about `_retracted` design

Because Doyle's `OUT` is *derived* and automatically restored, the project has no way — within Doyle's vocabulary — to express "this node was declared OUT by an authoritative source and must not silently come back `IN` when a local justification happens to become satisfied." That's the requirement `_retracted` was invented to satisfy:

- **`truth_value` is a projection of the justification graph** — c
