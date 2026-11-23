---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Function: _is_visible in reasons/api.py

## Purpose

`_is_visible` is a small, pure access-control predicate used across `reasons/api.py` to decide whether a belief node should be visible to a caller with a given set of access tags.

It exists to centralize the visibility rule:

> A node is visible if every tag in its `metadata["access_tags"]` is present in the caller’s `visible_to` list. Nodes with no access tags are public.

This is used both for filtering read/list/search/export results and for enforcing per-node access checks in functions like `show_node`, `explain_node`, and `trace_assumptions`.

## Contract

Preconditions:

- `node` must be a node-like object with a `.metadata` attribute that supports `.get(...)`.
- `node.metadata` is assumed to be a mapping, normally a `dict`.
- If `metadata["access_tags"]` exists, it is assumed to be an iterable of hashable values, normally strings.
- `visible_to` is expected to be a list of strings when passed directly to `_is_visible`.
- `visible_to=None` is **not** handled as “see everything” by this function. Callers must guard it.

Postconditions:

- Returns `True` if the node has no access tags, or if all of its access tags are contained in `visible_to`.
- Returns `False` if the node has at least one access tag not contained in `visible_to`.
- Does not mutate the node, metadata, or `visible_to`.

Important invariant:

- Visibility uses **all tags**, not any tag. A node tagged `["a", "b"]` requires both `"a"` and `"b"` in `visible_to`.

## Parameters

### `node`

The node object whose visibility is being tested. The function accesses:

```python
node.metadata.get("access_tags", [])
```

Expected shape:

```python
node.metadata = {
    "access_tags": ["team-a", "internal"],
    ...
}
```

If `access_tags` is missing, `None`, or an empty list, the node is treated as public.

This is duck-typed: there is no type annotation or runtime check requiring a specific `Node` class.

### `visible_to`

The caller’s granted access tags.

Examples:

```python
_is_visible(node_with_tags, ["team-a", "internal", "public"])
# True if node tags are ["team-a", "internal"]

_is_visible(node_with_tags, ["team-a"])
# False if node tags are ["team-a", "internal"]
```

Edge cases:

- `visible_to=[]` means the caller has no tags. Untagged nodes remain visible; tagged nodes are hidden.
- `visible_to=None` is not supported by this function if the node has non-empty tags. It will eventually call `set(None)` and raise `TypeError`.
- Duplicate tags in `visible_to` do not matter because it is converted to a set.
- Matching is exact, case-sensitive string membership. No normalization, prefix matching, wildcards, or hierarchy.

## Return Value

Returns a `bool`.

- `True` when:
  - `metadata["access_tags"]` is missing.
  - `metadata["access_tags"]` is empty or otherwise falsy.
  - Every tag in `access_tags` exists in `visible_to`.

- `False` when:
  - `access_tags` is non-empty and at least one tag is not in `visible_to`.

Callers use this result in two main ways:

- As a filter: skip the node if `False`.
- As a permission gate: raise `PermissionError` if `False`.

## Algorithm

Step by step:

1. Read the node’s access tags:

   ```python
   tags = node.metadata.get("access_tags", [])
   ```

2. If `tags` is falsy, immediately return `True`:

   ```python
   if not tags:
       return True
   ```

   This covers missing key, `None`, and `[]`.

3. Convert the caller’s granted tags into a set:

   ```python
   visible_set = set(visible_to)
   ```

4. Check that every node tag is present in the caller’s set:

   ```python
   return all(t in visible_set for t in tags)
   ```

   `all(...)` short-circuits on the first missing tag.

## Side Effects

None, apart from reading `node.metadata`.

Specifically:

- No database access.
- No file I/O.
- No logging.
- No mutation of `node`, `node.metadata`, or `visible_to`.
- Creates a temporary `set` for `visible_to` on every call.

The temporary `set` means repeated calls in large loops rebuild the same set each time. That is simple and safe, but could be optimized by callers if visibility checks become hot.

## Error Handling

The function does not explicitly validate inputs and does not catch exceptions.

Possible exceptions:

- `AttributeError` if `node` has no `.metadata`, or if `.metadata` is `None` and does not support `.get`.
- `TypeError` if `visible_to` is `None` and `tags` is non-empty, because `set(None)` fails.
- `TypeError` if `visible_to` contains unhashable values.
- `TypeError` if `tags` is truthy but not iterable.
- `TypeError` if an element of `tags` is unhashable, because `t in visible_set` requires `t` to be hashable.

The function does not raise `PermissionError` itself. Callers decide whether a `False` result means “skip this node” or “raise an access error.”

## Usage Patterns

Typical filtering usage:

```python
if visible_to is not None and not _is_visible(node, visible_to):
    continue
```

Typical permission-enforcement usage:

```python
if visible_to is not None and not _is_visible(node, visible_to):
    raise PermissionError(
        f"Node '{node_id}' requires access tags not in {visible_to}"
    )
```

Important caller obligation:

- `None` means “do not filter” at the API boundary, but it must not be passed into `_is_visible` for tagged nodes. This is why callers usually wrap calls with `visible_to is not None`.
- `[]` is meaningful: it means the caller has no access tags.
- A visible node can still reference antecedents that are invisible; this helper only checks the node’s own tags, not transitive dependency visibility.

## Dependencies

- No imports or external modules are used directly.
- It depends on the duck-typed node object shape, specifically `node.metadata`.
- The metadata is normally populated by `add_node(...)` and related import/export paths.
- The rule is exercised indirectly by many API functions, including `get_status`, `show_node`, `explain_node`, `trace_assumptions`, `trace_access_tags`, `export_network`, `export_markdown`, `export_card`, `compact`, `lookup`, `search`, `list_nodes`, and `list_gated`.

## Topics to Explore

- [file] `tests/test_api.py` — Existing coverage for visibility, access-tag filtering, and permission failures around `visible_to`.
- [file] `reasons/network.py` — Defines the node shape and metadata behavior that `_is_visible` relies on.
- [function] `reasons/api.py:export_network` — Shows how `_is_visible` affects exported nodes and whether references to hidden antecedents can still appear.
- [general] `access-tag-filtering-semantics` — Clarifies the distinction between `visible_to=None`, `visible_to=[]`, and non-empty tag lists across API call sites.
- [function] `reasons/api.py:trace_access_tags` — Handles transitive access-tag tracing, which is distinct from this node-local visibility check.

## Beliefs

- `access-tags-use-all-semantics` — `_is_visible` requires every node tag to be present in `visible_to`; a single missing tag hides the node.
- `untagged-nodes-are-unconditionally-visible` — Missing, `None`, or empty `access_tags` returns `True`, even when `visible_to=[]`.
- `visible-to-none-not-supported-with-tags` — `_is_visible` raises `TypeError` if `visible_to=None` and the node has non-empty tags; callers must guard `visible_to is not None`.
- `access-tag-matching-is-exact-and-case-sensitive` — Visibility uses set membership with no normalization, prefix matching, wildcards, or hierarchy.
- `visibility-check-has-no-side-effects` — `_is_visible` only reads `node.metadata` and returns a bool; it performs no mutation, I/O, or logging.
