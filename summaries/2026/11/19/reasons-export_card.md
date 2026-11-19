---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/export_card.py

# `reasons/export_card.py`

## Purpose

This module renders an in-memory `Network` as a single Markdown document intended to serve as a HuggingFace-compatible EEM card / `README.md`. It owns:

- metadata normalization via `build_meta`
- summary statistics
- derivation-depth calculation
- YAML frontmatter generation
- human-readable Markdown body generation

It is deliberately side-effect free: it returns a string and does not write files. Callers are responsible for persisting the result as `README.md`.

## Key Components

### `_node_depth(nid, nodes, memo=None) -> int`

Recursively computes the derivation depth of a node.

Contract:

- Missing node → depth `0`.
- Node with no `justifications` → depth `0`; these are treated as premises/observations.
- If `node.supporting_justification` is a valid index into `node.justifications`, only that justification is considered.
- Otherwise, all justifications are considered.
- A node’s depth is `1 + max(depth(antecedent))` over the selected justification antecedents.
- Results are memoized in `memo`.

Important implementation detail:

```python
memo[nid] = 0
```

is assigned before recursing. This acts as a cycle sentinel: re-entering the same node returns `0` instead of recursing forever. It does not validate acyclicity, and cyclic graphs can produce depths that are bounded but not necessarily meaningful.

Invalid `supporting_justification` values fall back to using all justifications because of:

```python
if si is not None and 0 <= si < len(node.justifications):
```

### `export_card(...) -> str`

Signature:

```python
export_card(
    network: Network,
    domain: list[str] | None = None,
    license: str = "mit",
    base_network: str | None = None,
    source_repos: list[str] | None = None,
) -> str
```

Builds the final Markdown card from the provided network.

Stats computed:

- `total = len(network.nodes)`
- `in_count`: nodes whose `truth_value == "IN"`
- `out_count = total - in_count`
- `premises`: nodes with no justifications
- `derived = total - premises`
- `nogood_count = len(network.nogoods)`
- `max_depth`: max `_node_depth(...)` across all nodes, defaulting to `0` for empty networks
- `retraction_pct`: `out_count / total * 100`, formatted to zero decimal places; `"0%"` when `total == 0`

Project name fallback:

```python
project_name = meta["project_name"] or "eem"
```

The card title is derived via:

```python
project_name.replace("-", " ").replace("_", " ").title()
```

Frontmatter fields emitted:

- `schema_version`
- `type: eem`
- `project_name`
- `domain` as YAML list or `[]`
- `license`
- `base_network` as quoted string or `null`
- `source_repos` as YAML list or `[]`
- `beliefs_total`
- `beliefs_in`
- `beliefs_out`
- `premises`
- `derived`
- `nogoods`
- `generator`

Body sections emitted:

- title
- `## Stats` table
- `## How to Use`
- `## Files`
- `## Quality`
- `## Limitations`
- `## License`

The function returns:

```python
"\n".join(lines)
```

Because `lines` ends with an empty string, the returned string ends with a trailing newline.

## Patterns

- **Pure rendering function**: `export_card` does not perform IO; it transforms structured data into text.
- **Template assembly**: frontmatter and body are built as lists of strings, then joined once.
- **Memoized recursive traversal**: `_node_depth` uses a shared `memo` dict for depth calculations.
- **Cycle sentinel**: assigning `memo[nid] = 0` before recursion prevents infinite recursion on cyclic networks.
- **Metadata normalization**: `build_meta` centralizes project/schema/generator metadata formatting.
- **Optional-field fallback**: `domain`, `base_network`, and `source_repos` all emit empty/null-ish YAML forms when absent.
- **Derived-document convention**: the file generates a GitHub/HuggingFace-style README with YAML frontmatter and Markdown body.

## Dependencies

Imports:

- `from .metadata import build_meta`
- `from .network import Network`

Expected `Network` shape:

- `network.meta`: dict-like metadata, containing at least optional `project_name` and `created_at`
- `network.nodes`: mapping of node IDs to node objects
- `network.nogoods`: sized collection

Expected node shape:

- `node.truth_value`
- `node.justifications`
- `node.supporting_justification`

Expected justification shape:

- `justification.antecedents`

Imported by:

- `reasons/api.py`
- `reasons_lib/export_card.py`
- `tests/test_export_card.py`

No third-party dependencies are used directly.

## Flow

1. Call `build_meta(...)` with `project_name`, `node_count`, and `created_at`.
2. Resolve fallback `project_name`.
3. Count total nodes, IN nodes, OUT nodes, premises, derived nodes, and nogoods.
4. Compute `max_depth` by running `_node_depth` for every node ID using a shared memo dict.
5. Compute retraction percentage with a guard for empty networks.
6. Build YAML frontmatter lines, including optional lists and null fields.
7. Append Markdown body sections: title, stats table, usage commands, file descriptions, quality summary, limitations, license.
8. Join all lines into one string and return it.

The only non-trivial data transformation is depth calculation. The rest is aggregation and string formatting.

## Invariants

- Every generated card starts with `---` and closes frontmatter with `---`, followed by a blank line.
- `derived` is always `total - premises`.
- `out_count` is always `total - in_count`.
- A node is considered IN only when `truth_value == "IN"` exactly. Any other truth value contributes to OUT.
- Nodes without justifications always have depth `0`.
- A valid `supporting_justification` index restricts depth traversal to that single justification.
- Invalid `supporting_justification` values fall back to all justifications.
- Empty networks are handled safely: percentage division is guarded and `max_depth` uses `default=0`.
- The function does not validate that the network is acyclic; the depth sentinel only prevents infinite recursion.
- YAML values are mostly interpolated directly. `project_name`, `schema_version`, and `base_network` are quoted, but `license`, `domain` entries, `source_repos`, and `generator` are not escaped.

## Error Handling

There is no explicit `try`/`except` in this module.

Error behavior:

- Missing or malformed node attributes will raise `AttributeError`.
- Passing wrong types for `domain`, `source_repos`, `license`, etc. will generally raise `TypeError` during string formatting or iteration.
- `_node_depth` gracefully handles missing node IDs and nodes without justifications.
- `_node_depth` gracefully ignores invalid `supporting_justification` indices.
- Empty networks avoid division by zero in retraction percentage calculation.
- Deep or pathological networks may hit Python recursion limits because `_node_depth` is recursive.
- `build_meta` errors propagate directly.
- No IO errors occur because the function never writes files.
- No explicit YAML validation occurs; unescaped values can produce malformed frontmatter.

## Topics to Explore

- [file] `reasons/metadata.py` — Shows exactly what `build_meta` returns, including `schema_version`, `generator`, and project-name normalization.
- [function] `reasons/export_card.py:_node_depth` — Worth inspecting for cycle behavior, `supporting_justification` semantics, and deep-network recursion limits.
- [file] `reasons/network.py` — Defines the `Network`, node, and justification data model that this exporter assumes.
- [file] `tests/test_export_card.py` — Captures expected frontmatter/body output and edge cases such as empty networks and domains.
- [file] `reasons/export_markdown.py` — Useful comparison for how the project distinguishes EEM card export from general Markdown export.

## Beliefs

- `export-card-returns-string-with-no-io` — `export_card` returns a single string and performs no file writes; callers persist the `README.md` themselves.
- `out-count-is-not-in` — A node is counted as OUT whenever its `truth_value` is anything other than the exact string `"IN"`.
- `premises-have-depth-zero` — `_node_depth` returns `0` for missing nodes, nodes without justifications, and premise nodes.
- `valid-supporting-justification-restricts-depth` — When `supporting_justification` is a valid index, only that justification’s antecedents contribute to the node’s derivation depth.
- `empty-network-is-safe` — `export_card` handles an empty network because retraction percentage guards `total == 0` and `max_depth` uses `default=0`.
