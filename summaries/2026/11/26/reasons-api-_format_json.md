---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Function: _format_json in reasons/api.py

# `_format_json`

## Purpose

`_format_json` is a private formatting helper used by `search()` when the caller requests JSON output. It takes:

- a `Network`-like object,
- an ordered list of matched node IDs,
- a set of related/neighbor node IDs,

and returns a single pretty-printed JSON string representing a flat array of result objects.

It exists to keep result formatting separate from search, filtering, and neighbor-expansion logic. Matched nodes are marked with `"match": true`; expanded neighbor nodes are marked with `"match": false` and `"relation": "neighbor"`.

## Contract

### Preconditions

- `net` must expose a `.nodes` mapping where `net.nodes[nid]` is a node-like object.
- Every ID in `matched_ids` and `neighbor_ids` must exist in `net.nodes`.
- `matched_ids` is expected to be a list of `str`.
- `neighbor_ids` is expected to be a set of `str`.
- The caller is expected to have already applied search-level filtering:
  - OUT filtering unless requested,
  - namespace filtering,
  - access-tag visibility filtering,
  - neighbor filtering.
- The caller is expected to ensure `neighbor_ids` is disjoint from `matched_ids`, though this function does not enforce or deduplicate overlap.

### Postconditions

- Returns a string containing a JSON array.
- Matched entries appear first, in the exact order of `matched_ids`.
- Neighbor entries appear afterward, sorted by node ID.
- Matched entries have `"match": true`.
- Neighbor entries have `"match": false` and `"relation": "neighbor"`.
- `net` is not mutated.

### Invariants

- Only direct lookups are performed: `net.nodes[nid]`.
- No additional filtering, validation, sorting of matched nodes, or deduplication occurs inside this function.
- The output is a JSON string, not a Python list or dict.

## Parameters

### `net`

A loaded network object, normally `reasons.network.Network`. The function relies on:

- `net.nodes`, a mapping from node ID to node object,
- `node.text`,
- `node.truth_value`,
- `node.source`.

No type checking is performed.

### `matched_ids: list[str]`

The node IDs that actually matched the search query.

- Order is preserved.
- Duplicates are not removed.
- Missing IDs raise `KeyError`.
- Entries are rendered with `"match": true`.

### `neighbor_ids: set[str]`

The related/neighbor node IDs discovered during search neighbor expansion.

- Set iteration is not preserved by definition, so the function calls `sorted(neighbor_ids)` for deterministic output.
- Should not contain IDs already in `matched_ids`; the caller in `search()` subtracts matched IDs before calling.
- Missing IDs raise `KeyError`.
- Entries are rendered with `"match": false` and `"relation": "neighbor"`.

## Return Value

Returns a string produced by:

```python
json.dumps(results, indent=2)
```

The string is a pretty-printed JSON array.

For example, each matched entry has the shape:

```json
{
  "id": "...",
  "text": "...",
  "truth_value": "IN",
  "source": "...",
  "match": true
}
```

Each neighbor entry has the shape:

```json
{
  "id": "...",
  "text": "...",
  "truth_value": "OUT",
  "source": "...",
  "match": false,
  "relation": "neighbor"
}
```

If both inputs are empty, it returns the JSON representation of an empty array, `[]`.

Callers receive a string, not structured data. They must either print it, return it directly from a CLI-facing API, or parse it if they need object access.

## Algorithm

1. Initialize `results = []`.
2. For each `nid` in `matched_ids`:
   - Look up `node = net.nodes[nid]`.
   - Append a dict containing `id`, `text`, `truth_value`, `source`, and `match: True`.
3. For each `nid` in `sorted(neighbor_ids)`:
   - Look up `node = net.nodes[nid]`.
   - Append a dict containing `id`, `text`, `truth_value`, `source`, `match: False`, and `relation: "neighbor"`.
4. Serialize `results` with `json.dumps(results, indent=2)` and return the resulting string.

The function does not sort `matched_ids`; it intentionally preserves whatever relevance or sort order the caller established. Neighbors are sorted only to make output stable because `neighbor_ids` is a set.

## Side Effects

None beyond reading from `net` and performing JSON serialization.

Specifically, it does not:

- open the database,
- mutate the network,
- write files,
- publish pub/sub events,
- update caches,
- modify `matched_ids` or `neighbor_ids`.

## Error Handling

No explicit error handling is present.

Possible errors:

- `KeyError` if any ID in `matched_ids` or `neighbor_ids` is missing from `net.nodes`.
- `TypeError` from `sorted(neighbor_ids)` if the set contains non-comparable or non-string values, which is why the type hint matters.
- `TypeError` from `json.dumps` if any node attribute included in the result is not JSON serializable.
- `AttributeError` if `net` or a node object lacks the expected attributes.

These exceptions propagate to the caller.

## Usage Patterns

The only call site in the provided file is inside `search()`:

```python
if format == "json":
    return _format_json(net, matched_ids, neighbor_ids)
```

That call happens after:

- matched IDs are found,
- OUT results are filtered unless `include_out=True`,
- namespace filtering is applied,
- visibility filtering is applied,
- tag filtering is applied,
- sorting is applied,
- pagination is applied,
- neighbor expansion is performed,
- neighbor IDs are filtered and subtracted from matched IDs.

So by the time `_format_json` sees the data, the caller has already made the important policy decisions. This helper is purely a serialization layer.

## Dependencies

- Standard library `json`.
- The node/network shape defined elsewhere, primarily:
  - `reasons/network.py` for `Network` and `Node`,
  - `net.nodes[nid].text`,
  - `net.nodes[nid].truth_value`,
  - `net.nodes[nid].source`.
- Called from `reasons/api.py:search`.

## Assumptions Not Enforced by Types

- `neighbor_ids` should be disjoint from `matched_ids`; the function does not deduplicate.
- All IDs should exist in `net.nodes`; direct indexing will fail otherwise.
- Node `.text`, `.truth_value`, and `.source` should be JSON-serializable in practice.
- The caller should have already applied access, truth-value, namespace, and tag filtering.
- Although the type hints say `list[str]` and `set[str]`, runtime enforcement is absent.
- The function assumes deterministic ordering for neighbors is desired, hence `sorted(neighbor_ids)`, but it deliberately preserves caller order for `matched_ids`.

## Topics to Explore

- [function] `reasons/api.py:search` — Shows the caller-side filtering, pagination, sorting, neighbor expansion, and dispatch to `_format_json`.
- [file] `reasons/network.py` — Defines the `Network` and node structures whose `text`, `truth_value`, and `source` attributes are read here.
- [function] `reasons/api.py:_format_minimal` — Useful comparison for how the same matched/neighbor data is rendered in another output mode.
- [general] `search-neighbor-expansion` — Explains how `neighbor_ids` is computed and why matched/neighbor disjointness matters.
- [file] `tests/test_api.py` — Shows expected API-level behavior and can reveal assumptions about JSON search output.

## Beliefs

- `_format-json-returns-json-string` — `_format_json` returns a pretty-printed JSON string, not a Python list or dict.
- `matched-ids-preserve-caller-order` — Matched result entries preserve the order of `matched_ids`, while neighbor entries are sorted by node ID.
- `neighbor-overlap-is-not-deduplicated` — The function assumes `neighbor_ids` is disjoint from `matched_ids`; overlapping IDs would produce duplicate result entries with different `match` flags.
- `json-formatting-requires-node-presence` — Direct indexing of `net.nodes[nid]` means any missing matched or neighbor ID raises `KeyError`.
- `search-applies-filters-before-json-formatting` — `search()` applies OUT, namespace, access-tag, and neighbor filtering before calling `_format_json`, so this helper does not enforce visibility or truth-value filtering itself.
