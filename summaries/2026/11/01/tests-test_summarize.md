---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_summarize.py

# `tests/test_summarize.py`

## Purpose

This file is the test suite for the **summarization** feature in the `reasons` project — a defeasible logic / argumentation system. It validates that `Network.summarize()` correctly creates a summary node that abstracts a group of nodes, that truth-value cascading propagates through the summary, that `compact()` output hides covered nodes behind their summary, and that the feature survives a save/load round-trip and the public API layer.

The docstring frames the feature in terms of **Doyle's SUM/GF (Summary/Generalization) justifications**: a summary node *replaces* a group of nodes in compact output rather than adding new reasoning. This is not a derivation — it's a presentation-layer abstraction backed by a structural link.

## Key Components

### `TestSummarizeBasic`
Tests the core `Network.summarize()` contract:

| Test | What it locks in |
|---|---|
| `test_create_summary` | `summarize()` returns `{"summary_id", "truth_value", "over"}`; the new node's `metadata["summarizes"]` records the covered IDs; initial truth value is `"IN"`. |
| `test_summary_depends_on_covered_nodes` | The summary gets an **SL** (Subsumption Link) justification whose antecedents are exactly the covered node IDs. This is the mechanism that makes truth-value cascading work. |
| `test_summary_out_when_covered_node_out` | Retracting *any* covered node flips the summary to `"OUT"` — the SL justification is all-antecedents-must-be-IN. |
| `test_summary_restored_when_covered_restored` | Re-asserting the retracted node restores the summary to `"IN"`. |
| `test_covered_nodes_get_metadata` | Each covered node's `metadata["summarized_by"]` list contains the summary ID (bidirectional link). |
| `test_nonexistent_node_raises` | Referencing a missing node in `over=[…]` raises `KeyError`. |
| `test_duplicate_summary_raises` | Calling `summarize()` with an already-used summary ID raises `ValueError`. |

### `TestSummarizeInCompact`
Tests the interaction between summaries and `compact(net, budget=…)`:

- **`test_summary_replaces_covered_in_compact`** — The summary appears in the output; covered nodes (`a`, `b`) are *hidden*; uncovered nodes (`c`) still appear.
- **`test_hidden_count_shown`** — The output includes a human-readable line containing `"hidden by summaries"`.
- **`test_no_summary_no_hiding`** — Without any summary, no hiding line appears and all nodes are shown.
- **`test_out_summary_does_not_hide`** — If the summary is `"OUT"` (because a covered node was retracted), the covered nodes are *not* hidden; node `b` still appears in the IN section.
- **`test_covers_count_shown`** — The summary line includes `"covers 2 nodes"` (the count of covered nodes).

### `TestSummarizePersistence`
- **`test_round_trip`** — Uses `tmp_path` to create a SQLite-backed `Storage`, saves a network with a summary, loads it back, and asserts: metadata survives, truth value is preserved, and **cascading still works after load** (`retract("a")` → summary goes `"OUT"`). This confirms the SL justification is serialized/deserialized correctly.

### `TestSummarizeAPI`
- **`test_summarize_api`** — Exercises the thin API wrapper `api.summarize()` and checks the returned dict shape.
- **`test_summarize_affects_compact`** — Confirms that `api.compact()` reflects the hiding behavior end-to-end.

## Patterns

- **Doyle-style justification types**: The system uses typed justifications (`SL`, `GF`, etc.). Summaries use `SL` (Subsumption Link), distinct from derivation links. The test at line ~28 (`test_summary_depends_on_covered_nodes`) pins down that `j.type == "SL"`.
- **Bidirectional metadata links**: `summarizes` (on the summary) ↔ `summarized_by` (on each covered node). This is a denormalized cross-reference stored in node metadata rather than a separate table.
- **Truth-value cascading via justifications**: The summary doesn't track covered nodes imperatively; instead the SL justification's antecedent list drives the IN/OUT propagation through the existing network engine.
- **Compact-output hiding**: The `compact()` function is responsible for *presentation* — it inspects `metadata["summarized_by"]` and suppresses covered nodes. The summary must be `"IN"` for hiding to take effect.
- **Layered testing**: Basic → Compact → Persistence → API. Each layer re-validates the feature through a different public surface, catching integration gaps.
- **`tmp_path` fixture** for all DB-backed tests, avoiding cross-test contamination.

## Dependencies

### Imports (this file depends on)
| Symbol | Source | Role |
|---|---|---|
| `pytest` | stdlib test framework | `pytest.raises`, fixtures |
| `Justification` | `reasons` | Constructed in setup to give node `c` an SL justification (test data only) |
| `Network` | `reasons.network` | The core graph; `add_node`, `summarize`, `retract`, `assert_node`, `.nodes` |
| `Storage` | `reasons.storage` | SQLite persistence for round-trip test |
| `compact` | `reasons.compact` | Budget-aware rendering with summary hiding |
| `api` | `reasons.api` | Thin function-based API over the same core |

### What depends on this file
Nothing imports a test file. This is a leaf in the dependency graph.

## Flow

Each test follows the same shape:

```
1. Build a Network (or init an API DB)
2. add_node(...) one or more times
3. net.summarize(id, label, over=[...])   ← the SUT
4. Assert on:
   - returned dict
   - net.nodes[id].truth_value
   - net.nodes[id].metadata["summarizes"]
   - net.nodes[covered].metadata["summarized_by"]
   - justification type & antecedents
5. (Optional) mutate: retract / assert_node
6. Assert cascaded truth value
7. (Optional) compact(net, budget=…) → string assertions
8. (Optional) Storage save → load → re-assert
```

The `TestSummarizeInCompact` tests add a string-level assertion layer on top, checking that `compact()` output contains/excludes specific substrings. The API tests replace steps 1-3 with `api.init_db` / `api.add_node` / `api.summarize`.

## Invariants

The tests collectively enforce these rules:

1. **Existence precondition**: Every ID in `over=[…]` must already exist in the network → `KeyError` otherwise.
2. **Uniqueness**: A summary ID can only be used once → `ValueError` on duplicate.
3. **Truth-value conjunction**: Summary is `"IN"` iff **all** covered nodes are `"IN"`. Any single retraction flips it to `"OUT"`; re-assertion restores `"IN"`.
4. **SL justification**: The summary must carry exactly one SL justification whose antecedents are the covered nodes. This is the *sole* mechanism for cascading.
5. **Bidirectional metadata**: `summarizes` and `summarized_by` must be consistent mirrors.
6. **Hiding only when IN**: `compact()` hides covered nodes *only* when the summary is `"OUT"` — no, only when the summary is `"IN"`. An OUT summary does **not** suppress its covered nodes.
7. **Persistence fidelity**: After `Storage.save()` → `Storage.load()`, all of the above invariants must still hold, including cascading behavior.

## Error Handling

- **`KeyError`** — raised by `Network.summarize()` when an ID in `over` is not a known node (line ~68, `test_nonexistent_node_raises`). This is a hard precondition failure; the test asserts it is raised, not caught.
- **`ValueError`** — raised when the summary ID already exists in the network (line ~72, `test_duplicate_summary_raises`).
- No error paths are tested for the `compact()` or `api.summarize()` layers. The API test only exercises the happy path. There's no test for, e.g., `api.summarize()` with a missing node ID, which would presumably propagate the `KeyError` from `Network`.

A developer adding new error cases (e.g., summarizing an empty `over` list, summarizing a node that is itself a summary) would need to add corresponding tests here.

## Topics to Explore

- [function] `reasons/network.py:summarize` — The core implementation: how it creates the SL justification, sets bidirectional metadata, and validates preconditions. Understanding this is prerequisite to understanding every test in this file.
- [file] `reasons/compact.py` — The compact renderer that reads `summarized_by` metadata to decide hiding, formats the `"hidden by summaries"` and `"covers N nodes"` lines, and handles the OUT-summary edge case.
- [file] `reasons/storage.py` — How justifications (including SL) and node metadata are serialized to and deserialized from SQLite. The round-trip test depends on this being lossless.
- [file] `reasons/api.py` — The thin API layer that `TestSummarizeAPI` exercises. Understanding how it delegates to `Network.summarize()` and wraps the result dict.
- [general] `doyle-argumentation-sum-gf` — The theoretical grounding: Doyle's SUM and GF justification types. The codebase distinguishes SL (used here) from other types; reading the project's README or docs for the justification-type taxonomy will clarify design decisions.

## Beliefs

- `summary-truth-requires-all-covered-in` — A summary node's truth value is IN if and only if every node in its `over` list is IN; a single retracted covered node causes the summary to go OUT.
- `sl-justification-drives-cascading` — The summary's truth-value propagation is entirely mediated by its SL justification's antecedent list; there is no separate "watcher" or subscription mechanism.
- `compact-hiding-requires-summary-in` — `compact()` suppresses covered nodes from output only when the covering summary is IN; an OUT summary does not hide its covered nodes.
- `bidirectional-metadata-links` — `summarizes` on the summary node and `summarized_by` on each covered node are maintained as consistent mirrors; both must survive a Storage save/load cycle.
- `summarize-enables-compact-abstraction` — Without a summary node, `compact()` has no hiding behavior; the summary is the sole mechanism by which `compact()` reduces visible node count via grouping.
