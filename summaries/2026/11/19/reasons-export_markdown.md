---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/export_markdown.py

# `reasons/export_markdown.py`

## Purpose

This module owns the export half of the beliefs.md round-trip: it serializes an in-memory `Network` into a deterministic, beliefs.md-compatible markdown string. It performs no file I/O and does not mutate the network; callers are responsible for writing the returned string.

The output contains:

1. YAML frontmatter generated through `build_meta`.
2. A `# Belief Registry` title and generated-file warning.
3. Optional `## Repos` section.
4. `## Claims` section with one subsection per node.
5. Optional `## Nogoods` section.

It flattens justification-level structure into node-level `Depends on`, `Unless`, and `Defeated by` bullets, so it is intentionally not a full object dump of every justification.

---

## Key Components

### `export_markdown(network: Network, repos: dict[str, str] | None = None) -> str`

The sole public function.

Contract:

- Reads `network.meta`, `network.nodes`, and `network.nogoods`.
- Reads each node’s `id`, `text`, `source`, `source_url`, `source_hash`, `date`, `truth_value`, `metadata`, and `justifications`.
- Reads each justification’s `antecedents` and `outlist`.
- Reads each nogood’s `id`, `discovered`, `resolution`, and `nodes`.
- Returns the complete markdown document as a single string.
- Does not write files and does not mutate the network.

### `build_meta(...)`

Called with:

```python
build_meta(
    project_name=network.meta.get("project_name", ""),
    node_count=len(network.nodes),
    created_at=network.meta.get("created_at", ""),
)
```

The returned dictionary must provide at least:

- `schema_version`
- `project_name`
- `updated_at`
- `node_count`
- `generator`

Those are interpolated directly into YAML frontmatter.

### Expected object shape

From this file’s perspective:

- `network.meta` is a dict-like object with `.get`.
- `network.nodes` is a dict mapping node ID to node object.
- `network.nogoods` is an iterable of nogood objects.
- A node has `id`, `text`, `source`, `source_url`, `source_hash`, `date`, `truth_value`, `metadata`, and `justifications`.
- A justification has `antecedents` and `outlist`, both iterables of node IDs.
- A nogood has `id`, `discovered`, `resolution`, and `nodes`.

---

## Patterns

### Pure serializer / string builder

The function accumulates a `lines: list[str]` and returns `"\n".join(lines)`. That makes the output easy to test and avoids side effects.

### Deterministic ordering

The document structure is ordered explicitly:

- Repos are sorted by repo name.
- Claims are sorted with IN nodes first, then all non-IN nodes.
- Within each group, nodes are sorted alphabetically by `node.id`.
- Nogoods are not sorted; they preserve `network.nogoods` order.

The claim sort key is:

```python
key=lambda n: (n.truth_value != "IN", n.id)
```

So despite the docstring saying “IN first, then OUT”, the implementation actually places any truth value other than `"IN"` after IN nodes.

### Metadata-as-extension-point

Only curated metadata fields are serialized, not arbitrary metadata. Important fields include:

- `beliefs_type`
- `source_type`
- `stale_reason`
- `retract_reason`
- `duplicate_of`
- `superseded_by`
- `accepted_pr`
- `defeats_node`
- `defeater_type`
- `defeat_reason_type`

This keeps the markdown schema stable.

### Flattened justification representation

Multiple justifications are merged into node-level bullets:

- `Depends on` is the de-duplicated union of all justification antecedents.
- `Unless` is the de-duplicated union of outlist nodes that are not classified as defeaters.
- `Defeated by` is the de-duplicated union of outlist nodes whose referenced node metadata says `defeats_node == node.id`.

This means per-justification grouping is lost in the markdown export.

### Defeater classification by back-reference

For every outlist ID `o`, the exporter looks up `network.nodes.get(o)`. It classifies `o` as a defeater only if:

```python
o_node.metadata.get("defeats_node") == node.id
```

Otherwise, or if `o` does not resolve to a node, it is emitted under `Unless`.

---

## Dependencies

### Imports

- `from .metadata import build_meta` — builds frontmatter metadata.
- `from .network import Network` — type annotation and source of the exported data model.

### Imported by

- `reasons/api.py`
- `reasons_lib/export_markdown.py` (parallel library copy)
- Tests including:
  - `tests/test_export_markdown.py`
  - `tests/test_metadata.py`
  - `tests/test_metadata_extended.py`
  - `tests/test_roundtrip.py`
  - `tests/test_source_type.py`

So this module is part of the public export path and is directly covered by round-trip and metadata tests.

---

## Flow

1. Build metadata via `build_meta`.
2. Emit YAML frontmatter:
   - `schema_version` quoted.
   - `project_name` quoted if non-empty, otherwise bare `project_name:`.
   - `updated_at` quoted.
   - `node_count` numeric.
   - `generator` unquoted.
3. Emit `# Belief Registry` and an HTML comment warning that the file is generated.
4. If `repos` is truthy, emit `## Repos` with sorted `- name: path` lines.
5. Emit `## Claims`.
6. Sort nodes by `(truth_value != "IN", id)`.
7. For each node:
   - Set `status = node.truth_value`.
   - If `status == "OUT"` and metadata has `stale_reason` or `retract_reason`, change `status` to `"STALE"`.
   - Choose `btype` from metadata `beliefs_type`, defaulting to:
     - `"DERIVED"` if the node has justifications.
     - `"OBSERVATION"` otherwise.
   - Emit header:

     ```markdown
     ### <node.id> [<status>] <btype>
     ```

   - Emit `node.text`.
   - Emit optional metadata bullets:
     - `Source`
     - `Source URL`
     - `Source type`
     - `Source hash`
     - `Date`
   - Walk all justifications:
     - Add unique antecedents to `all_deps`.
     - Walk `j.outlist`.
     - If the outlist node exists and has `metadata["defeats_node"] == node.id`, add it to `all_defeaters` with its defeater type and reason type.
     - Otherwise, add it to `all_unless` if not already present.
   - Emit `Depends on`, `Unless`, and `Defeated by` bullets as applicable.
   - Emit `Stale reason` only if status is `"STALE"` or `"OUT"` and a retract/stale reason exists.
   - Emit `Duplicate of`, `Superseded by`, and `Accepted PR` bullets when present.
   - Emit a blank line.
8. If `network.nogoods` is non-empty:
   - Emit `## Nogoods`.
   - For each nogood, emit `### <id>`, optional `Discovered` and `Resolution`, and always `Affects: ...`.
9. Return `"\n".join(lines)`.

The returned string normally ends with a newline because the builder appends blank strings at section boundaries.

---

## Invariants

- Section order is fixed: frontmatter, title/comment, optional Repos, Claims, optional Nogoods.
- Claim order is deterministic: IN nodes first, then all other truth values, alphabetical by ID within each group.
- Node header format is exactly:

  ```markdown
  ### {node.id} [{status}] {btype}
  ```

- `status` comes from `node.truth_value`, except that `OUT` plus a stale/retract reason becomes `STALE`.
- `beliefs_type` defaults to `DERIVED` when justifications exist, otherwise `OBSERVATION`.
- `Depends on` is derived from antecedents across all justifications and de-duplicated while preserving encounter order.
- `Unless` and `Defeated by` are derived from justification outlists.
- An outlist ID is a defeater only when the referenced node’s metadata says `defeats_node == current_node.id`.
- Unresolved outlist IDs are treated as `Unless`, not as errors.
- Nogoods preserve network order.
- No escaping is performed for node text, IDs, reason strings, repo paths, or metadata strings. Values containing newlines or markdown syntax can affect the rendered document.

---

## Error Handling

There is no explicit `try`/`except`.

- Missing optional metadata is handled through `.get(...)` and truthiness checks.
- An empty `project_name` produces a bare `project_name:` YAML key.
- Missing outlist target nodes do not raise; they are rendered as `Unless` entries.
- Missing `stale_reason`/`retract_reason` simply omits the stale reason bullet.
- Missing `discovered` or `resolution` on a nogood omits those bullets.

Errors are propagated to the caller for contract violations such as:

- `network` lacking `meta`, `nodes`, or `nogoods`.
- A node lacking expected attributes such as `metadata` or `justifications`.
- A justification lacking `antecedents` or `outlist`.
- A nogood lacking `nodes`.
- Non-string values in lists passed to `", ".join(...)`.

The function does no validation of duplicate IDs, dangling references, cycles, markdown safety, or schema correctness beyond what is needed to build the string.

---

## Topics to Explore

- [file] `reasons/import_beliefs.py` — Counterpart parser; compare what this exporter writes with what the importer can reconstruct.
- [function] `reasons/metadata.py:build_meta` — Defines the frontmatter defaults, including `schema_version`, `updated_at`, and `generator`.
- [file] `reasons/network.py` — Defines the `Network`/node/justification objects whose attributes this exporter reads.
- [file] `tests/test_export_markdown.py` — Shows exact expected markdown, edge cases, and regression coverage.
- [general] `beliefs-markdown-schema` — Status, belief type, defeater, stale, duplicate, and nogood conventions used in the exported format.

## Beliefs

- `export-markdown-is-pure-string-export` — `export_markdown` returns a markdown string and performs no file I/O or network mutation.
- `export-markdown-sorts-in-nodes-before-non-in-nodes` — Claims are sorted with `truth_value == "IN"` first, then all non-IN nodes, alphabetically by node ID within each group.
- `export-markdown-maps-out-with-stale-or-retract-reason-to-stale` — A node with truth value `OUT` and metadata `stale_reason` or `retract_reason` is rendered with status `STALE`.
- `export-markdown-classifies-defeaters-via-defeats-node-metadata` — An outlist ID is rendered as `Defeated by` only when the referenced node’s metadata has `defeats_node` equal to the current node ID; otherwise it is rendered under `Unless`.
- `export-markdown-aggregates-justification-antecedents` — `Depends on` is the de-duplicated union of antecedents across all of a node’s justifications, preserving first-seen order.
