---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: _format_node in reasons/build_wiki.py

## `_format_node`

### Purpose

`_format_node` renders a single belief node as a Markdown subsection for the static wiki. It is the deterministic fallback renderer used by `build_wiki` when no LLM-generated page content exists for a topic group. It includes the node header, truth status, duplicate/superseded metadata, detected defeaters, node text, dependency links, and dependent/support links.

It exists so wiki pages can still be generated without an LLM while preserving cross-references between belief nodes.

---

### Contract

**Preconditions**

- `node_id` is a string-like identifier.
- `node_detail` is a mapping for one belief node and must contain:
  - `"truth_value"`
  - `"text"`
- `node_to_page` maps node IDs to page filenames, e.g. `{node_id: "topic-page.md"}`.
- If `all_details` is provided and truthy, it must map node IDs to their detail mappings.
- Any `justifications`, `dependents`, `outlist`, `antecedents`, etc. fields, when present, are assumed iterable in the way the code uses them. In practice they are assumed to be lists or list-like containers.

**Postconditions**

- Returns a Markdown string for one belief node.
- Does not mutate inputs.
- Does not perform I/O.
- Missing page mappings do not raise; they fall back to raw node IDs.
- Missing defeater details in `all_details` are silently skipped.

**Important invariants**

- Cross-reference links are emitted as `[id](page#id)` only when `node_to_page.get(id)` returns a truthy page value.
- Otherwise the raw node ID is emitted.
- Defeater rendering only occurs when `all_details` is truthy.
- Antecedents are split into “active” and “other” only when `supporting_justification` is a valid index into `justifications`.

---

### Parameters

#### `node_id`

The belief ID being rendered. It is used in the section header and as the link text/anchor for cross-references.

#### `node_detail`

A mapping describing the node. The function directly indexes:

```python
node_detail['truth_value']
node_detail['text']
```

So missing either raises `KeyError`.

It uses `.get()` for optional fields such as:

- `metadata`
- `justifications`
- `supporting_justification`
- `dependents`

The code assumes values like `justifications` and `dependents` are iterable containers, not `None`.

#### `node_to_page`

A lookup from node ID to Markdown page filename. Example:

```python
{
    "belief-a": "topic-one.md",
    "belief-b": "topic-two.md",
}
```

If a node ID is absent or maps to a falsey page, the formatter emits the raw ID instead of a link.

#### `all_details=None`

Optional full mapping of node IDs to node details. It is only used to resolve defeater metadata from entries in each justification’s `outlist`.

If `all_details` is `None` or otherwise falsey, defeater rendering is skipped entirely. If it is truthy, outlist nodes are looked up in it; missing entries are ignored.

---

### Return Value

Returns a string produced by:

```python
return "\n".join(lines)
```

The string contains a `### {node_id}` heading and zero or more Markdown paragraphs. The function appends a final empty line before joining, so the returned string generally ends with a trailing newline.

The caller is responsible for placing this string into a larger page. `build_wiki` appends it to `page_lines` and later joins the whole page.

---

### Algorithm

1. Start a `lines` list and append:
   - `### {node_id}`
   - `**Status:** {node_detail['truth_value']}`
   - a blank line.

2. Read `metadata = node_detail.get("metadata") or {}` and render duplicate/superseded metadata:
   - If `"duplicate_of"` exists, link to that canonical node if possible.
   - If `"superseded_by"` exists, link to that new node if possible.
   - Links use `page#node_id`; otherwise the raw ID is used.
   - Each relationship is followed by a blank line.

3. If `all_details` is truthy, scan the current node’s justifications and their `outlist` entries:
   - For each outlist node `o`, look up `o` in `all_details`.
   - If found, check whether its metadata has `defeats_node == node_id`.
   - If so, collect `(o, o_meta, o_detail.get("text", ""))`.
   - For each collected defeater, render:
     - `d_type = metadata.get("defeater_type", "defeater")`
     - `r_type = metadata.get("defeat_reason_type", "")`
     - label is `"{d_type}, {r_type}"` if `r_type` exists, otherwise just `d_type`.
   - The defeater’s `text` is collected as `d_text` but is not used in the output.
   - Defeaters are not sorted or deduplicated.

4. Append the node’s text and a blank line.

5. Determine antecedents:
   - Read `justifications`.
   - Read `supporting_justification`.
   - If `supporting_justification` is a valid index into `justifications`:
     - The designated justification’s antecedents become the active set.
     - Antecedents from other justifications that are not already active become `other_antecedents`.
   - Otherwise:
     - All antecedents from all justifications become active.
     - `other_antecedents` remains empty.

6. Render dependencies:
   - If there are active antecedents, render them sorted and linked where possible.
     - Label is `**Depends on (active):**` only when there are also `other_antecedents`.
     - Otherwise label is `**Depends on:**`.
   - If there are `other_antecedents`, render them sorted and linked where possible under `**Depends on (other):**`.

7. Render dependents/supports:
   - Read `dependents`.
   - If non-empty, sort and render them under `**Supports:**`.

8. Append a final blank line and return the joined Markdown string.

---

### Side Effects

None. The function is pure with respect to its arguments and performs no file I/O, network calls, or global state mutation.

It does create temporary sets and lists, but does not mutate `node_detail`, `node_to_page`, or `all_details`.

---

### Error Handling

There is no explicit `try`/`except`.

Potential failures:

- `KeyError` if `node_detail["truth_value"]` or `node_detail["text"]` is missing.
- `TypeError` if optional fields are present but not iterable, e.g.:
  - `justifications=None`
  - `dependents=None`
  - `outlist=None`
  - `antecedents=None`
- `AttributeError` if a justification entry is not a mapping but the code calls `.get()` on it.
- `TypeError` if `supporting_justification` is non-numeric but passes the comparison expression in an incompatible way.
- `TypeError` if `supporting_justification` is a float such as `0.0`, because `justifications[si]` requires an integer index.
- `TypeError` at final join if any appended element is not a string, such as `node_detail["text"] = None`.

Silently tolerated cases:

- A node ID missing from `node_to_page` produces a raw ID instead of a link.
- An outlist node missing from `all_details` is ignored.
- An outlist node whose metadata does not contain `defeats_node == node_id` is ignored.
- An invalid `supporting_justification` index falls back to treating all antecedents as active.

---

### Usage Patterns

`_format_node` is called by `build_wiki` in the fallback path:

```python
for nid in sorted(nids):
    detail = node_details.get(nid)
    if detail:
        page_lines.append(
            _format_node(
                nid,
                detail,
                node_to_page,
                all_details=node_details,
            )
        )
```

So in normal use:

- `node_detail` comes from `node_details[nid]`.
- `node_to_page` is precomputed from the topic grouping.
- `all_details` is the full `node_details` mapping so defeater metadata can be resolved.

The caller must ensure the detail mapping contains the required `"truth_value"` and `"text"` keys and that optional fields have the expected container shapes.

---

### Dependencies

This function depends only on Python built-ins and the expected shape of belief detail dictionaries.

It does not call other helpers in `reasons/build_wiki.py`, including `_linkify`. It builds its own cross-reference links directly.

The broader module depends on filesystem and LLM-related helpers, but `_format_node` itself is self-contained.

---

### Assumptions Not Enforced by the Type System

- `node_detail` contains `"truth_value"` and `"text"` as strings.
- `metadata`, `justifications`, and their nested entries are dictionaries.
- `justifications`, `outlist`, `antecedents`, and `dependents` are iterable containers, typically lists.
- `supporting_justification` is an integer or `None`.
- Node IDs are safe enough to use as Markdown link text and anchors.
- Page filenames in `node_to_page` are valid Markdown link targets.
- Anchors like `page#node_id` correspond to actual rendered anchors.
- `all_details` is a non-empty mapping when defeater rendering is desired.
- Defeater metadata stores `defeats_node` as a scalar matching `node_id`, not a list of IDs.

---

## Topics to Explore

- [file] `reasons/build_wiki.py` — Shows how `_format_node` is used during fallback Markdown generation and how `node_to_page` is constructed.
- [function] `reasons/build_wiki.py:_format_beliefs_for_prompt` — Contrasts deterministic node rendering with the LLM prompt-oriented belief formatter.
- [function] `reasons/build_wiki.py:_linkify` — Handles cross-page linking after page assembly and is useful to compare with `_format_node`’s direct link generation.
- [general] `belief-node-detail-shape` — The formatter relies heavily on fields like `truth_value`, `justifications`, `supporting_justification`, `dependents`, and `metadata`.
- [file] `tests/test_build_wiki.py` — Likely verifies expected Markdown output, cross-links, and edge cases for this formatter.

## Beliefs

- `format-node-requires-truth-value-and-text` — `_format_node` directly indexes `truth_value` and `text`, so missing either raises `KeyError`.
- `format-node-renders-defeaters-only-with-all-details` — Defeater rendering is skipped when `all_details` is `None` or otherwise falsey.
- `format-node-links-use-page-anchors` — Cross-reference links are emitted as `[node_id](page#node_id)` when `node_to_page` has a truthy page for that ID; otherwise the raw ID is emitted.
- `supporting-justification-splits-active-and-other-antecedents` — When `supporting_justification` is a valid index, that justification’s antecedents are active and non-overlapping antecedents from other justifications are rendered as “other”.
- `format-node-has-no-defeater-sorting` — Defeaters are emitted in outlist iteration order and are not deduplicated.
