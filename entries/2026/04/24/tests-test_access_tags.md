# File: tests/test_access_tags.py

**Date:** 2026-04-24
**Time:** 17:03

## Purpose

`tests/test_access_tags.py` is the test suite for the access control system built on **data source provenance tags** (issue #38). It verifies that when beliefs are derived from restricted data sources (e.g., finance records, HR data), the restriction tags propagate through the dependency graph and are enforced at the API boundary. This is the security boundary test — it ensures that an agent with only `["public"]` clearance cannot read, explain, trace, or export beliefs tainted by `["finance"]` data.

The file owns two concerns: (1) tag **inheritance** through the belief network, and (2) tag **enforcement** (filtering and denial) across every API surface.

## Key Components

### `TestInheritance` — Tag propagation through the dependency graph

Tests the `Network`-level mechanics of how `access_tags` flow from parent nodes to derived nodes:

- **Single parent** (`test_derived_inherits_single_parent_tags`): A node justified by one tagged parent inherits that parent's tags.
- **Union of parents** (`test_derived_inherits_union_of_parents`): A node justified by two differently-tagged parents gets the union — `["finance", "hr"]`.
- **Merge with explicit tags** (`test_derived_merges_explicit_and_inherited_tags`): A derived node with its own tags merges them with inherited ones.
- **Chain inheritance** (`test_chain_inheritance`): Tags propagate transitively: `a → b → c` all carry `["finance"]`.
- **Diamond inheritance** (`test_diamond_inheritance`): The classic diamond `a,b → m1,m2 → c` produces the union without duplicates.
- **No tags, no inheritance** (`test_no_tags_no_inheritance`): Untagged parents produce untagged children — the system doesn't fabricate tags.
- **Dynamic updates** (`test_add_justification_updates_tags`, `test_add_justification_propagates_tags_to_dependents`): Adding a new justification to an existing node triggers tag recomputation that cascades to all downstream dependents.
- **Storage round-trip** (`test_tags_persist_through_storage`): Tags survive SQLite serialization/deserialization via the `api` layer.

### `TestVisibleTo` — API-level access enforcement

Tests that every API function that returns node data respects `visible_to` filtering:

- **`list_nodes`**: Filters out nodes whose tags aren't a subset of the caller's clearance. Untagged nodes are always visible. Multi-tag nodes require the caller to have *all* tags (subset semantics).
- **`show_node`**: Raises `PermissionError` when the caller lacks clearance.
- **`lookup` / `search`**: Text searches exclude forbidden nodes from results, including neighbor expansion.
- **`get_status`**: The status summary only counts visible nodes.
- **`explain_node`**: Blocks explanation of forbidden nodes. Crucially, tag inheritance means a derived node inherits its parent's restriction, so you can't leak a restricted antecedent by explaining its child.
- **`trace_assumptions`**: Same inheritance-based blocking; the caller can't trace through a restricted subgraph.
- **`export_network`**: Filters both nodes and nogoods (a nogood referencing a forbidden node is itself hidden).
- **`export_markdown` / `compact`**: Markdown and summary outputs exclude forbidden nodes.

### `TestTraceAccessTags` — Diagnostic tag tracing

Tests `Network.trace_access_tags()` and its API wrapper `api.trace_access_tags()`, which answer the question "what tags does this node carry, considering its full ancestry?" Covers premises, chains, diamonds, missing nodes (`KeyError`), and permission checks on the trace endpoint itself.

## Patterns

1. **Fixture-based DB isolation**: The `db_path` fixture creates a fresh temp directory per test. `TestInheritance` and `TestTraceAccessTags` mostly operate on in-memory `Network` objects directly; only the persistence and API tests use the DB fixture.

2. **Two-layer testing**: Inheritance tests hit `Network` directly (unit-level), while `TestVisibleTo` and some `TestTraceAccessTags` tests go through `api.*` (integration-level). This mirrors the architecture: `Network` owns propagation logic, `api` owns enforcement.

3. **Subset semantics for multi-tag nodes**: A node tagged `["finance", "hr"]` requires the caller to have *both* tags. This is a deliberate design choice tested in `test_list_nodes_multi_tag_requires_all` — it's the secure default (conjunction, not disjunction).

4. **`PermissionError` as the denial signal**: The API raises `PermissionError` (not returning `None` or an empty dict) when a specific node is requested but forbidden. List-style endpoints silently filter instead.

## Dependencies

**Imports:**
- `pytest` — test framework and fixtures
- `reasons_lib.Justification` — dataclass for SL justifications (antecedents + outlist)
- `reasons_lib.network.Network` — the in-memory belief graph with propagation
- `reasons_lib.api` — the functional API layer that wraps Network + SQLite storage

**Imported by:** Nothing — this is a leaf test module.

## Flow

The typical test flow is:

1. Build a small network (2–5 nodes) with explicit `access_tags` on premises.
2. Add derived nodes with `Justification(type="SL", antecedents=[...])`.
3. Assert that `metadata["access_tags"]` on derived nodes equals the expected union.
4. For API tests: call an API function with `visible_to=[...]` and assert the result includes/excludes the right nodes, or raises `PermissionError`.

The `add_justification_propagates_tags_to_dependents` test is notable for its flow: it builds a chain `a → b → c` with no tags, then retrofits a tagged justification onto `a` and verifies the tags cascade to `b` and `c`.

## Invariants

1. **Tag inheritance is the union of all antecedent tags** across all justifications, merged with any explicit tags on the node itself.
2. **Untagged nodes are always visible** — `visible_to` only restricts tagged nodes.
3. **Multi-tag nodes require superset clearance** — `visible_to` must contain *all* of the node's tags.
4. **Single-node lookups raise `PermissionError`** when forbidden; collection endpoints silently filter.
5. **Tag propagation is transitive and updates dynamically** — adding a justification recomputes tags for the target and all its dependents.
6. **Tag inheritance prevents information leakage via derived nodes** — you can't explain or trace a node that inherits a restriction you lack.

## Error Handling

- **`PermissionError`**: Raised by `show_node`, `explain_node`, `trace_assumptions`, and `trace_access_tags` when `visible_to` doesn't cover the node's tags. Tests verify this with `pytest.raises(PermissionError)`.
- **`KeyError`**: Raised by `Network.trace_access_tags()` for nonexistent node IDs (`test_missing_node_raises`).
- No error swallowing — every error path is explicitly tested.

## Topics to Explore

- [file] `reasons_lib/network.py` — Contains `add_node`, `add_justification`, and `trace_access_tags` — the actual propagation logic these tests verify
- [function] `reasons_lib/api.py:list_nodes` — The `visible_to` filtering implementation and how subset semantics are enforced at the storage/query level
- [function] `reasons_lib/network.py:add_justification` — How tag recomputation cascades to dependents when a new justification is added
- [diff] `a7ef621` — The commit that introduced access tags (PR #42) — shows the full implementation these tests were written against
- [general] `subset-vs-intersection-semantics` — The choice that multi-tag nodes require superset clearance is a security design decision worth understanding for future tag-related features

## Beliefs

- `access-tags-union-inheritance` — A derived node's `access_tags` is the sorted, deduplicated union of all antecedent tags across all justifications, merged with any explicit tags on the node itself.
- `visible-to-superset-semantics` — A node with `access_tags: ["finance", "hr"]` is only visible to callers whose `visible_to` list contains both `"finance"` and `"hr"` (superset check, not intersection).
- `untagged-always-visible` — Nodes with no `access_tags` key in metadata are never filtered by `visible_to` — they are treated as public.
- `single-node-api-raises-permissionerror` — API functions that target a single node by ID (`show_node`, `explain_node`, `trace_assumptions`, `trace_access_tags`) raise `PermissionError` when the caller lacks clearance, rather than returning empty results.
- `tag-propagation-is-dynamic` — Adding a justification to an existing node triggers tag recomputation that cascades through all downstream dependents via BFS.

