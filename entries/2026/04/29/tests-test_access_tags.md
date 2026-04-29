# File: tests/test_access_tags.py

**Date:** 2026-04-29
**Time:** 17:09

## Purpose

`test_access_tags.py` is the test suite for **access control via data-source provenance tags** (issue #38). It validates that beliefs in the TMS can carry `access_tags` metadata — strings like `"finance"` or `"hr"` — and that these tags flow correctly through the belief network to enforce visibility restrictions. The file owns two distinct concerns:

1. **Tag inheritance** — when a derived belief is justified by tagged premises, it automatically inherits the union of their tags.
2. **Visibility filtering** — API operations (`list_nodes`, `show_node`, `search`, `explain`, `export`, etc.) respect a `visible_to` parameter, hiding beliefs the caller isn't authorized to see and raising `PermissionError` for direct access attempts.

## Key Components

### `TestInheritance` (10 tests)

Tests the in-memory `Network` layer. All tests construct a `Network()` directly, add nodes with explicit `access_tags` in metadata, and assert that derived nodes pick up the right tags.

- **Single-parent inheritance**: a derived node gets its sole parent's tags.
- **Union of parents**: multiple antecedents → sorted union of all tags (no duplicates implied by the assertions).
- **Merge with explicit tags**: if a derived node declares its own tags AND has tagged parents, both sets merge.
- **Chain inheritance**: tags propagate transitively through `a → b → c`.
- **Diamond inheritance**: `a` and `b` feed into `m1`/`m2` which feed into `c` — `c` gets the union `["finance", "hr"]`.
- **No-tag passthrough**: if no ancestor has tags, derived nodes don't get an `access_tags` key at all.
- **`add_justification` live update**: adding a new justification to an existing node re-computes its tags and propagates downstream.
- **Storage round-trip**: tags survive write-to-SQLite and read-back via `api.add_node` / `api.show_node`.

### `TestVisibleTo` (21 tests)

Tests the `api` module's filtering layer. Uses a `db_path` fixture (temp SQLite) and exercises every API endpoint that accepts `visible_to`:

| API function | Behavior tested |
|---|---|
| `list_nodes` | Filters out nodes whose tags aren't a subset of `visible_to`; untagged nodes always pass |
| `show_node` | Raises `PermissionError` when caller lacks required tags |
| `lookup` / `search` | Text queries exclude forbidden nodes from results |
| `get_status` | Summary counts only include visible nodes |
| `explain_node` | Raises if the target or any antecedent is forbidden; returns full trace when allowed |
| `trace_assumptions` | Same gate-then-filter pattern |
| `export_network` | Strips forbidden nodes AND nogoods that reference them |
| `export_markdown` | Markdown output omits forbidden beliefs |
| `compact` | Compacted output excludes forbidden beliefs |

### `TestTraceAccessTags` (7 tests)

Tests `Network.trace_access_tags()` and `api.trace_access_tags()` — a utility that walks the justification graph upward and collects the full set of tags a node is subject to, even those inherited from distant ancestors. Also checks that `KeyError` is raised for missing nodes and `PermissionError` for forbidden API-level access.

## Patterns

**Two-layer testing**: `TestInheritance` tests the `Network` (in-memory graph) directly, while `TestVisibleTo` and `TestTraceAccessTags` test through the `api` module (which wraps Network + SQLite storage). This mirrors the project's architecture: network logic is pure, API adds persistence and access control.

**Fixture-based DB isolation**: the `db_path` fixture creates a fresh temp directory per test, so each test gets its own SQLite file with no cross-contamination.

**Raises-based access control**: forbidden access is tested with `pytest.raises(PermissionError)`, establishing that the API uses exceptions (not empty results) for direct-access violations, but uses filtering (empty/reduced results) for list operations.

**Subset semantics for `visible_to`**: a node's tags must be a subset of the caller's `visible_to` set. `test_list_nodes_multi_tag_requires_all` is the critical test — a node tagged `["finance", "hr"]` is invisible to a caller with only `["finance"]`.

## Dependencies

**Imports**:
- `pytest` — test framework
- `reasons_lib.Justification` — the justification dataclass (SL type, antecedents, outlist)
- `reasons_lib.network.Network` — in-memory belief graph
- `reasons_lib.api` — public API layer with persistence and access control

**Imported by**: nothing (it's a test file).

## Flow

Most tests follow this pattern:

1. Create nodes (some tagged, some not) via `Network.add_node()` or `api.add_node()`.
2. Create derived nodes with `Justification(type="SL", antecedents=[...])`.
3. Assert that tag inheritance produced the expected `metadata["access_tags"]` on derived nodes.
4. For visibility tests: call an API function with `visible_to=[...]` and assert which nodes appear/disappear, or that `PermissionError` fires.

The `add_justification_propagates_tags_to_dependents` test is the most complex flow: it builds an untagged chain `a → b → c`, then retroactively adds a tagged justification to `a` and asserts the tags cascade down to `b` and `c`.

## Invariants

1. **Tag inheritance is transitive**: if any ancestor has tags, all descendants inherit them.
2. **Inheritance is union-based**: the derived node's effective tags are the union of all antecedents' tags plus any explicit tags.
3. **Untagged nodes are always visible**: `visible_to` filtering never hides nodes without `access_tags`.
4. **Subset gate**: a tagged node is visible only if its tags are a subset of the caller's `visible_to` set.
5. **Direct access raises, list access filters**: `show_node` and `explain_node` raise `PermissionError`; `list_nodes`, `search`, `export_*` silently exclude.
6. **No tags key when empty**: premises without tags don't have an `access_tags` key in metadata at all (not an empty list).
7. **Retroactive propagation**: adding a justification to an existing node re-computes tags for the entire downstream subgraph.

## Error Handling

- `PermissionError` is the single exception type for access violations, raised by `show_node`, `explain_node`, `trace_assumptions`, and `api.trace_access_tags` when `visible_to` doesn't cover the node's tags.
- `KeyError` is raised by `Network.trace_access_tags()` for nonexistent node IDs.
- No error swallowing — the tests assert that these exceptions propagate cleanly.

## Topics to Explore

- [file] `reasons_lib/network.py` — Contains `Network.add_node`, `add_justification`, and `trace_access_tags` — the core inheritance logic under test
- [function] `reasons_lib/api.py:list_nodes` — Implements the `visible_to` filtering that most `TestVisibleTo` tests exercise
- [function] `reasons_lib/network.py:add_justification` — The retroactive tag propagation path tested in `test_add_justification_propagates_tags_to_dependents`
- [general] `subset-vs-intersection-semantics` — The choice that a node's tags must be a *subset* of `visible_to` (not just intersect) has major UX implications — worth understanding the design rationale from issue #38
- [file] `reasons_lib/storage.py` — How `access_tags` are serialized/deserialized in SQLite, which the storage round-trip test depends on

## Beliefs

- `access-tags-subset-gate` — A tagged node is visible only when its `access_tags` are a subset of the caller's `visible_to` set; partial overlap is insufficient
- `untagged-nodes-always-visible` — Nodes without `access_tags` metadata pass all `visible_to` filters and are never hidden
- `tag-inheritance-is-transitive-union` — Derived nodes inherit the sorted union of all ancestor tags, and this propagates through arbitrarily long justification chains
- `direct-access-raises-list-access-filters` — API functions for single-node access (`show_node`, `explain_node`, `trace_assumptions`) raise `PermissionError` on forbidden nodes, while list/export functions silently exclude them
- `add-justification-propagates-tags-downstream` — Calling `add_justification` on an existing node recomputes access tags for the target and all its transitive dependents

