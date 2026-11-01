---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_access_tags.py

# `tests/test_access_tags.py`

## Purpose

This file is the test suite for **access control via provenance tags** (issue #38) in the `ftl-reasons` logical inference engine. It validates a security model where beliefs (nodes in a defeasible-logic network) carry `access_tags` that propagate *downward* through derivations, and where the public API enforces visibility by requiring callers to present the full set of tags they're authorized for.

In short: it owns the contract that **a derived belief is at least as restricted as the most-restricted premise it depends on**.

## Key Components

### `TestInheritance` — Tag propagation rules (in-memory `Network`)

These tests exercise `Network.add_node` and `Network.add_justification` directly, without persistence:

| Test | Contract verified |
|------|-------------------|
| `test_derived_inherits_single_parent_tags` | A node with a justification whose antecedent has tags `["finance"]` gets `access_tags=["finance"]` in its metadata. |
| `test_derived_inherits_union_of_parents` | Multiple antecedents → union of all their tags. |
| `test_derived_merges_explicit_and_inherited_tags` | If a node is *explicitly* given tags at creation **and** inherits from antecedents, both sets are merged. |
| `test_premise_does_not_inherit` | A leaf premise with no justifications never gains `access_tags` unless explicitly set. |
| `test_chain_inheritance` | Tags propagate transitively: A→B→C all end up with `["finance"]`. |
| `test_diamond_inheritance` | Diamond DAG: the union is computed correctly without duplication. |
| `test_no_tags_no_inheritance` | If no antecedent has tags, the derived node has **no** `access_tags` key at all (not an empty list). |
| `test_add_justification_updates_tags` | Adding a *new* justification to an existing node re-computes its tag set. |
| `test_add_justification_propagates_tags_to_dependents` | Mutating a node's antecedents retroactively pushes new tags to all downstream dependents. |
| `test_tags_persist_through_storage` | Tags survive a write→read round-trip via the `api` (SQLite) layer. |

### `TestVisibleTo` — API-level filtering (persisted via `db_path`)

These tests exercise the `reasons.api` module. The key parameter is `visible_to: list[str]`:

- **Filter semantics**: a node is visible **only if** the caller's `visible_to` set is a *superset* of the node's `access_tags`. Untagged nodes are always visible.
- **Methods covered**: `list_nodes`, `show_node`, `lookup`, `search`, `search_neighbors` (implicitly), `get_status`, `explain_node`, `trace_assumptions`, `export_network`, `export_markdown`, `compact`.
- **Error semantics**: `show_node`, `explain_node`, `trace_assumptions`, and `trace_access_tags` raise `PermissionError` when the caller lacks a required tag. List-style methods (`list_nodes`, `search`, `lookup`) silently omit restricted nodes.
- **Nogood filtering**: `test_export_network_filters_nogoods` verifies that a nogood containing any restricted node is hidden from unauthorized callers.

### `TestTraceAccessTags` — Full tag computation

Tests `Network.trace_access_tags(node_id)` and `api.trace_access_tags(node_id)`:

- Walks the **entire** ancestor closure (not just direct antecedents) and returns the union of all tags encountered.
- Returns `[]` for untagged nodes (note the contrast with `add_node`, which omits the key entirely).
- Raises `KeyError` for a nonexistent node.
- The API variant raises `PermissionError` if `visible_to` doesn't cover the target's tags.

## Patterns

1. **Two-tier testing**: In-memory `Network` tests (fast, no I/O) verify the *algorithm*; `api` + `db_path` tests verify *persistence* and the *public contract*. This mirrors the repo's dual structure (`reasons/` vs `reasons_lib/`).

2. **Fixture isolation**: `db_path` uses pytest's `tmp_path` fixture so every test gets a fresh SQLite file — no cross-test contamination.

3. **Superset authorization check**: The visibility rule is `set(node.access_tags).issubset(visible_to)`. This is an **all-of** (AND) semantic, not any-of (OR). `test_list_nodes_multi_tag_requires_all` makes this explicit.

4. **Union-based inheritance**: Tags accumulate as a union along the derivation DAG. There is no subtraction or negation.

5. **Key-absence vs empty-list distinction**: Internally, "no tags" means the `access_tags` key is **absent** from the metadata dict (see `test_premise_does_not_inherit`, `test_no_tags_no_inheritance`). The `trace_access_tags` API, however, returns `[]` in that case. This is an intentional boundary translation.

6. **Retroactive propagation**: `test_add_justification_propagates_tags_to_dependents` shows that mutating the graph *after* initial creation triggers a cascading re-computation. The implementation must be reactive, not one-shot.

## Dependencies

**Imports (this file → project):**
- `reasons.Justification` — dataclass for a defeasible rule (type, antecedents, outlist, label)
- `reasons.network.Network` — the in-memory inference graph; provides `add_node`, `add_justification`, `trace_access_tags`, `nodes`
- `reasons.api` — the persistence-backed public API; provides `add_node`, `show_node`, `list_nodes`, `search`, `lookup`, `explain_node`, `trace_assumptions`, `export_network`, `export_markdown`, `compact`, `get_status`, `trace_access_tags`, `add_nogood`
- `pytest` — test runner and fixture system

**Imported by:**
- Nothing imports this file (it's a leaf test module). It is discovered and executed by pytest via `pyproject.toml` / `conftest.py`.

## Flow

```
Premise A (access_tags=["finance"])
    │
    │  Justification SL (antecedents=["a"])
    ▼
Derived B (inherits ["finance"])
    │
    │  Justification SL (antecedents=["b"])
    ▼
Derived C (inherits ["finance"])
```

1. **Tag assignment**: A premise is created with `metadata={"access_tags": [...]}` (or via `api.add_node(access_tags=[...])`).
2. **Tag inheritance on derivation**: When `Network.add_node` is called with `justifications=[...]`, the implementation collects `access_tags` from every antecedent node, unions them with any explicitly provided tags, and writes the result into the new node's metadata.
3. **Retroactive re-computation**: `Network.add_justification` re-evaluates the target node's tag set and then walks all transitive dependents, updating their tags as well.
4. **Visibility check at the API layer**: Every read method (`list_nodes`, `show_node`, `search`, etc.) accepts an optional `visible_to` list. Before returning a node, it checks `set(node.metadata.get("access_tags", [])) ⊆ set(visible_to)`. Failing nodes are omitted (list APIs) or raise `PermissionError` (single-node APIs).
5. **Trace**: `trace_access_tags` performs a full DFS/BFS over the ancestor subgraph, collecting the union of all `access_tags` encountered, and returns the flattened list.

## Invariants

- **Monotonic restriction**: A derived node's tag set is always a superset of each antecedent's tag set. Tags can only *add* restrictions, never remove them.
- **No upward leakage**: Tags never flow from consequent to antecedent. A premise's tags are immutable unless explicitly set.
- **Key absence = public**: A node without an `access_tags` key is visible to *everyone*, regardless of `visible_to`.
- **All-tags-required (AND)**: A caller must hold *every* tag on a node, not just one.
- **Nogood atomicity**: A nogood is hidden if *any* of its constituent nodes is restricted.
- **Storage fidelity**: Tags written to SQLite must be read back identically (round-trip invariant).

## Error Handling

| Condition | Behavior | Where tested |
|-----------|----------|--------------|
| Caller lacks a required tag on a **single-node** API (`show_node`, `explain_node`, `trace_assumptions`, `trace_access_tags`) | Raises `PermissionError` | `test_show_node_raises_on_forbidden`, `test_explain_raises_on_forbidden`, `test_trace_raises_on_forbidden`, `test_api_trace_access_tags_raises_on_forbidden` |
| Caller lacks a tag on a **collection** API (`list_nodes`, `search`, `lookup`, `export_*`, `compact`) | Node is silently omitted from results | `test_list_nodes_filters_by_access`, `test_search_respects_visible_to`, etc. |
| `trace_access_tags` called on a nonexistent node (in-memory) | Raises `KeyError` | `test_missing_node_raises` |
| `visible_to` not provided (None) | All nodes visible, no filtering | `test_list_nodes_visible_to_none_shows_all`, `test_show_node_no_filter` |

There is no "swallow and return empty" path for single-node lookups — the design is deliberately loud (`PermissionError`) so that a client can distinguish "not found" from "forbidden."

## Topics to Explore

- [file] `reasons/network.py` — Contains the `Network` class with `add_node`, `add_justification`, and `trace_access_tags`; understanding the tag-inheritance algorithm and the retroactive propagation logic lives here.
- [file] `reasons/api.py` — The public API that threads `visible_to` through every read method; this is where `PermissionError` is raised and where the superset check is implemented.
- [file] `reasons/storage.py` — Handles SQLite serialization/deserialization of node metadata; needed to understand how `access_tags` survive the round-trip and how `conftest.py` fixtures wire up the DB.
- [function] `reasons/network.py:trace_access_tags` — The DFS/BFS that walks the ancestor closure; critical for understanding performance characteristics on deep or wide DAGs.
- [general] `issue-38-access-control` — The original GitHub issue and PR that introduced this feature; reviewing the discussion will clarify why AND-semantics (all tags required) was chosen over OR-semantics, and what the intended threat model is.

## Beliefs

- `derived-node-tags-are-superset-of-antecedent-tags` — A node's effective `access_tags` always contain the union of every ancestor's tags; restrictions only accumulate, never diminish.
- `visible-to-uses-and-semantics` — A caller must hold *every* tag on a node to view it; holding a subset is insufficient (verified by `test_list_nodes_multi_tag_requires_all`).
- `single-node-apis-raise-permissionerror` — `show_node`, `explain_node`, `trace_assumptions`, and `trace_access_tags` raise `PermissionError` on access denial, while collection APIs (`list_nodes`, `search`) silently omit restricted nodes.
- `untagged-nodes-are-always-visible` — A node whose metadata lacks the `access_tags` key passes every `visible_to` filter, including empty lists.
- `add-justification-triggers-retroactive-propagation` — Mutating a node's antecedents via `add_justification` re-computes tags on that node *and* cascades the update through all transitive dependents, not just the target.
