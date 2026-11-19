---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_access_tags.py

# `tests/test_access_tags.py`

## Purpose

This is a pytest contract/specification file for issue #38: access control based on data-source provenance tags. It tests two layers of the `reasons` package:

1. **Graph-level tag inheritance** in `reasons.network.Network`: derived nodes inherit `access_tags` from their antecedents.
2. **API-level access enforcement** in `reasons.api`: operations such as listing, showing, searching, explaining, tracing, exporting, and compacting must respect a caller’s `visible_to` tags.

The file is not production code. It defines expected behavior that `reasons/network.py`, `reasons/api.py`, and the storage layer must satisfy.

---

## Key Components

### Fixture

```python
@pytest.fixture
def db_path(tmp_path):
    return str(tmp_path / "test.db")
```

Provides an isolated SQLite database path per test using pytest’s `tmp_path`. API integration tests use this so they do not share state.

---

### `TestInheritance`

Tests in-memory `Network` behavior around `metadata["access_tags"]`.

Contracts verified:

- `test_derived_inherits_single_parent_tags`  
  A derived node with one SL justification inherits the antecedent’s `access_tags`.

- `test_derived_inherits_union_of_parents`  
  A derived node with multiple antecedents gets the union of antecedent tags:  
  `["finance"]` + `["hr"]` → `["finance", "hr"]`.

- `test_derived_merges_explicit_and_inherited_tags`  
  Explicit node tags are merged with inherited tags.

- `test_premise_does_not_inherit`  
  A plain premise added without tags does **not** get an `access_tags` key in `metadata`.

- `test_chain_inheritance`  
  Tags propagate transitively: `a → b → c`; `c` receives `a`’s tags through `b`.

- `test_diamond_inheritance`  
  Diamond-shaped dependency graphs produce the union of all contributing antecedent tags.

- `test_no_tags_no_inheritance`  
  If no upstream node has tags, derived nodes do not gain an empty `access_tags` key.

- `test_add_justification_updates_tags`  
  `Network.add_justification()` must recompute a node’s inherited tags. Adding a second justification from an `hr`-tagged node to an already `finance`-tagged derived node produces `["finance", "hr"]`.

- `test_add_justification_propagates_tags_to_dependents`  
  Adding a tagged justification to an existing node must propagate new tags to its dependents. This verifies dynamic recalculation, not just inheritance at initial insertion.

- `test_tags_persist_through_storage`  
  `api.add_node(..., access_tags=["finance"])` and `api.add_node(..., sl="a")` persist tags through storage. `api.show_node()` returns `metadata["access_tags"]` for both the premise and the derived node.

---

### `TestVisibleTo`

Tests API access filtering via the `visible_to` argument. The visibility rule implied by the tests is:

> A node is visible if it has no `access_tags`, or if **all** of its `access_tags` are contained in `visible_to`.  
> Omitting `visible_to` disables filtering.

Contracts verified:

- `test_list_nodes_filters_by_access`  
  `api.list_nodes(visible_to=["finance"])` includes untagged nodes and finance-tagged nodes, excludes HR-tagged nodes.

- `test_list_nodes_visible_to_none_shows_all`  
  Omitting `visible_to` returns all nodes.

- `test_list_nodes_untagged_always_visible`  
  Untagged nodes remain visible regardless of `visible_to`.

- `test_list_nodes_subset_match`  
  A node tagged `["finance"]` is visible to `visible_to=["finance", "hr"]`.

- `test_list_nodes_multi_tag_requires_all`  
  A node tagged `["finance", "hr"]` is **not** visible to `visible_to=["finance"]`; it requires both tags.

- `test_show_node_raises_on_forbidden`  
  `api.show_node("fin", visible_to=["hr"])` raises `PermissionError`.

- `test_show_node_allowed` / `test_show_node_no_filter`  
  Authorized access and no-filter access both succeed.

- `test_lookup_respects_visible_to`  
  `api.lookup()` excludes invisible matches.

- `test_search_respects_visible_to`  
  `api.search()` excludes invisible matches.

- `test_search_neighbors_filtered`  
  Search/neighbor expansion must not leak hidden antecedent data. A node derived from a finance-tagged secret also inherits the restriction, so the secret is not returned.

- `test_status_respects_visible_to`  
  `api.get_status()` filters `nodes` and reports `total` based on visible nodes only.

- `test_explain_raises_on_forbidden`  
  `api.explain_node()` raises `PermissionError` for an invisible node.

- `test_explain_inherits_restriction_from_antecedent`  
  Inheritance prevents explanation leakage. A derived node inherits its restricted antecedent’s tags, so a caller without those tags cannot explain the derived node. With sufficient tags, the explanation includes both derived and antecedent nodes.

- `test_trace_raises_on_forbidden`  
  `api.trace_assumptions()` raises `PermissionError` for an invisible target.

- `test_trace_filters_premises`  
  With sufficient visibility, trace returns visible premises. With insufficient visibility, the target is rejected because it inherited the restricted tag.

- `test_export_network_respects_visible_to`  
  `api.export_network()` filters invisible nodes out of `data["nodes"]`.

- `test_export_network_filters_nogoods`  
  A nogood referencing an invisible node is removed. It appears only when all referenced nodes are visible.

- `test_export_markdown_respects_visible_to`  
  `api.export_markdown()` excludes invisible nodes from generated markdown.

- `test_compact_respects_visible_to`  
  `api.compact()` excludes invisible nodes from compact output.

---

### `TestTraceAccessTags`

Tests direct access-tag tracing on `Network` and through the API.

Contracts verified:

- `Network.trace_access_tags("a")` returns `["finance"]` for a tagged premise.
- Returns `[]` for an untagged node.
- Collects all tags through a chain: `c` returns `["finance", "hr"]`.
- Collects the union across a diamond graph.
- Raises `KeyError` for a nonexistent node.
- `api.trace_access_tags("b", db_path=...)` returns:
  ```python
  {
      "node_id": "b",
      "access_tags": ["finance"],
  }
  ```
- `api.trace_access_tags("a", visible_to=["hr"])` raises `PermissionError` when the target is not visible.

---

## Patterns

- **Pytest class grouping by concern**: inheritance, visibility enforcement, and trace access tags are separated into `TestInheritance`, `TestVisibleTo`, and `TestTraceAccessTags`.
- **Fixture-based isolation**: `db_path` uses `tmp_path` so each API test gets a fresh SQLite database.
- **Arrange–Act–Assert**: most tests add nodes, invoke an API method, then assert returned data or raised exceptions.
- **Dual-layer testing**:
  - `Network` tests exercise graph behavior directly in memory.
  - `api` tests exercise persistence and public API enforcement.
- **Black-box contract testing**: tests mostly rely on public methods (`add_node`, `add_justification`, `show_node`, `list_nodes`, `search`, `explain_node`, etc.) rather than internal storage details.
- **Security-oriented assertions**: tests explicitly verify that derived nodes cannot leak restricted antecedents through explanation, tracing, search, export, or compact output.
- **Guard/filter semantics**: `visible_to=None` or omitted means “no filtering”; `visible_to=[...]` applies the tag subset rule.

---

## Dependencies

### Imports

- `pytest` — test runner, fixtures, and `pytest.raises`.
- `reasons.Justification` — used to construct graph justifications with:
  ```python
  Justification(type="SL", antecedents=[...], outlist=[], label="")
  ```
- `reasons.network.Network` — in-memory graph used for inheritance and trace tests.
- `reasons.api` — public API used for storage-backed visibility tests.

### Depends on

This test file depends on behavior implemented in:

- `reasons/network.py` — `Network.add_node`, `Network.add_justification`, `Network.trace_access_tags`, and node metadata handling.
- `reasons/api.py` — `add_node`, `show_node`, `list_nodes`, `lookup`, `search`, `get_status`, `explain_node`, `trace_assumptions`, `trace_access_tags`, `export_network`, `export_markdown`, `compact`, and `add_nogood`.
- The storage layer behind `db_path` — likely SQLite-backed persistence.

### Imported by

This is a pytest test module. Production code does not import it. Pytest collects it as `tests/test_access_tags.py`.

Note: the repository contains both `reasons/` and `reasons_lib/`. This test imports from `reasons`, not `reasons_lib`.

---

## Flow

1. **In-memory inheritance flow**
   - Create `Network()`.
   - Add nodes via `net.add_node(...)`, optionally with `metadata={"access_tags": [...]}`.
   - Add derived nodes with `justifications=[Justification(...)]`.
   - `Network` computes and stores inherited tags under `metadata["access_tags"]`.
   - `net.add_justification(...)` must recompute tags for that node and propagate changes to dependents.

2. **Storage-backed API flow**
   - `db_path` creates a temporary SQLite path.
   - `api.add_node(...)` writes premise/derived nodes and metadata.
   - APIs receive `visible_to=[...]`.
   - Listing/search/export/compact-style APIs silently filter out invisible nodes.
   - Direct-access APIs like `show_node`, `explain_node`, `trace_assumptions`, and `trace_access_tags` raise `PermissionError` when the target is invisible.

3. **Trace flow**
   - `Network.trace_access_tags(node_id)` traverses the dependency graph and returns the union of tags from the node and its antecedents.
   - `api.trace_access_tags(...)` performs visibility checking first, then returns a structured dictionary containing `node_id` and `access_tags`.

---

## Invariants

- Access tags are stored under `node.metadata["access_tags"]`.
- Derived-node tags are the **union** of:
  - the node’s explicit `access_tags`, and
  - all antecedent tags reachable through its justifications.
- Adding a new justification can increase the tag set of a node and its dependents.
- A premise added without tags has no `"access_tags"` key in metadata.
- `Network.trace_access_tags()` returns `[]` for an untagged node.
- Untagged nodes are always visible.
- A tagged node is visible only when its full tag set is a subset of `visible_to`.
- Omitting `visible_to` disables filtering.
- `show_node`, `explain_node`, `trace_assumptions`, and `api.trace_access_tags` raise `PermissionError` for invisible targets.
- `list_nodes`, `lookup`, `search`, `get_status`, `export_network`, `export_markdown`, and `compact` filter invisible data without raising.
- `export_network` filters nodes first, then removes nogoods that reference any invisible node.
- Multi-tag list comparisons in tests use exact list equality, e.g. `["finance", "hr"]`, so tag ordering must remain deterministic.

---

## Error Handling

- `PermissionError` is the expected signal for unauthorized direct access:
  - `api.show_node`
  - `api.explain_node`
  - `api.trace_assumptions`
  - `api.trace_access_tags`
- `KeyError` is expected from `Network.trace_access_tags("nonexistent")`.
- Filtering APIs do not raise; they return only visible data:
  - `list_nodes`
  - `lookup`
  - `search`
  - `get_status`
  - `export_network`
  - `export_markdown`
  - `compact`
- Tests use `pytest.raises(...)` to assert error behavior.

---

## Topics to Explore

- [file] `reasons/network.py` — Implements node insertion, justification handling, tag inheritance, propagation, and `trace_access_tags`.
- [function] `reasons/network.py:add_justification` — Responsible for recomputing tags and propagating them to dependents.
- [function] `reasons/api.py:list_nodes` — Central example of `visible_to` filtering semantics, including untagged-node visibility and subset matching.
- [function] `reasons/api.py:explain_node` — Shows how inherited restrictions prevent leaking restricted antecedents.
- [general] `access-control-tag-visibility-semantics` — The subset rule, omitted-filter behavior, and silent-filter vs `PermissionError` split across API methods.

---

## Beliefs

- `derived-nodes-union-access-tags` — A derived node’s `access_tags` is the union of its explicit tags and all antecedent tags across its justifications, and adding a justification can propagate new tags to dependents.
- `visible-to-subset-semantics` — A tagged node is visible only when all of its tags are contained in `visible_to`; untagged nodes are always visible and omitting `visible_to` disables filtering.
- `restricted-show-explain-trace-raise-permissionerror` — `show_node`, `explain_node`, `trace_assumptions`, and `api.trace_access_tags` raise `PermissionError` when the requested node is not visible.
- `listing-search-export-compact-filter-silently` — `list_nodes`, `lookup`, `search`, `get_status`, `export_network`, `export_markdown`, and `compact` exclude invisible data without raising.
- `export-network-drops-restricted-nogoods` — `export_network` removes a nogood when any referenced node is invisible to the caller.
