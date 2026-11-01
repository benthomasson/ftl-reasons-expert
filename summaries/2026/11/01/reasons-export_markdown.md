---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/export_markdown.py

# `reasons/export_markdown.py`

## Purpose

This module is a **serializer**: it converts an in-memory `Network` object (the project's core reasoning graph) into a flat, human-readable Markdown document with YAML frontmatter. It is the "to disk" direction of the roundtrip — the inverse of whatever `import_beliefs` or similar does. The output is intentionally constrained to the "beliefs.md" schema so that the file can be version-controlled, reviewed in pull requests, and re-imported without loss of semantic meaning.

It owns exactly one responsibility: **deterministic rendering of a Network into a specific Markdown dialect**. It does not write files, hit the network, or mutate the `Network`.

## Key Components

### `export_markdown(network: Network, repos: dict[str, str] | None = None) -> str`

The sole public function. Contract:

| Aspect | Detail |
|--------|--------|
| **Input** | A fully-constructed `Network` (nodes dict, nogoods list, meta dict) and an optional `repos` mapping. |
| **Output** | A single `str` — the complete Markdown document including frontmatter. |
| **Side effects** | None. Pure read of `network` state. |
| **Raises** | Nothing explicitly; will `KeyError` if `network.nodes` is missing a node referenced in a justification's antecedents (the code guards this with `.get()` only for defeater lookup, not for `all_deps`). |

### Internal sub-concerns (not separate functions, but logical blocks):

1. **Frontmatter generation** — delegates to `build_meta()` for schema version, timestamps, node count, generator string.
2. **Status mapping** — translates internal `truth_value` (`IN`/`OUT`) + metadata flags into the external status labels (`IN`, `OUT`, `STALE`).
3. **Beliefs type inference** — if `node.metadata["beliefs_type"]` is absent, defaults to `"DERIVED"` (has justifications) or `"OBSERVATION"` (no justifications).
4. **Dependency flattening** — iterates all justifications, collects unique antecedents into `all_deps`, splits outlist entries into `all_unless` vs. `all_defeaters` based on the `defeats_node` metadata pointer.
5. **Nogoods rendering** — appends a `## Nogoods` section only if `network.nogoods` is non-empty.

## Patterns

- **Pure function / no I/O.** The caller is responsible for writing the returned string to a file or stdout. This makes the function trivially testable (see `tests/test_export_markdown.py`).
- **Accumulator list.** All output lines are appended to a single `lines: list[str]` and joined at the end. This avoids f-string complexity and makes section ordering explicit.
- **Stable sort with composite key.** `key=lambda n: (n.truth_value != "IN", n.id)` — a boolean tuple that places `IN` nodes (False = 0) before `OUT` nodes (True = 1), then breaks ties alphabetically by `node.id`. This guarantees deterministic output for diff-friendliness.
- **Optional-section guards.** Repos, Nogoods, and per-node metadata lines are only emitted when the underlying data is present. This keeps the output minimal.
- **Deduplication via membership check.** `all_deps`, `all_unless`, and `all_defeaters` each guard against duplicates with `if x not in ...` before appending. With potentially many justifications per node, this is O(n²) in the worst case but node counts are expected to be small.
- **Mirror module.** `reasons_lib/export_markdown.py` is a byte-identical copy (the repo ships both a `reasons` package and a `reasons_lib` package, likely for different install contexts or backward compatibility).

## Dependencies

### Imports (this file depends on)

| Import | Role |
|--------|------|
| `from .metadata import build_meta` | Produces the YAML frontmatter dict (schema_version, updated_at, node_count, generator). |
| `from .network import Network` | Type annotation only — the function reads `network.nodes`, `network.nogoods`, `network.meta`. |

### Imported by (what depends on this file)

| Consumer | Why |
|----------|-----|
| `reasons/api.py` | Public API surface — exposes `export_markdown` as a callable endpoint. |
| `reasons_lib/export_markdown.py` | Mirror copy (same logic, different package namespace). |
| `tests/test_export_markdown.py` | Unit tests for the full output. |
| `tests/test_metadata.py`, `tests/test_metadata_extended.py` | Verify frontmatter fields. |
| `tests/test_roundtrip.py` | Import → export → import cycle. |
| `tests/test_source_type.py` | Verifies the `source_type` metadata line renders. |

## Flow

```
export_markdown(network, repos)
│
├─ 1. Call build_meta(...) → meta dict
│
├─ 2. Emit YAML frontmatter (--- block)
│
├─ 3. Emit "# Belief Registry" header + comment
│
├─ 4. IF repos: emit "## Repos" bullet list (sorted by key)
│
├─ 5. Emit "## Claims" header
│
├─ 6. Sort nodes:  key = (truth_value != "IN", id)
│
├─ 7. FOR each node:
│     ├─ Determine status: IN | OUT | STALE
│     ├─ Determine btype:  metadata or inferred
│     ├─ Emit "### {id} [{status}] {btype}"
│     ├─ Emit node.text
│     ├─ Emit source/source_url/source_type/source_hash/date (if present)
│     ├─ Walk node.justifications:
│     │   ├─ Collect antecedents → all_deps (deduped)
│     │   └─ Walk j.outlist:
│     │       ├─ IF o_node.metadata["defeats_node"] == node.id → all_defeaters
│     │       └─ ELSE → all_unless
│     ├─ Emit "Depends on" / "Unless" / "Defeated by" lines
│     ├─ Emit stale-reason, duplicate_of, superseded_by, accepted_pr (if present)
│     └─ Blank line
│
├─ 8. IF network.nogoods: emit "## Nogoods" section
│
└─ 9. return "\n".join(lines)
```

The data transformation is strictly **many-to-one**: a graph (nodes + directed justification edges + nogoods) is flattened into a linear document. No information is lost, but the graph structure is re-encoded as per-node "Depends on" lists rather than as explicit edge objects.

## Invariants

1. **Deterministic ordering.** The output is fully determined by `(truth_value, id)` sort. Two calls with the same `Network` produce byte-identical strings (barring `updated_at` in frontmatter, which depends on wall-clock).
2. **IN-before-OUT.** The sort key `(n.truth_value != "IN", n.id)` guarantees every `IN` node appears before every `OUT`/`STALE` node.
3. **Status promotion rule.** An `OUT` node is displayed as `STALE` *only if* it carries `stale_reason` or `retract_reason` in metadata. A bare `OUT` node stays `OUT`.
4. **Defeater disambiguation.** An outlist entry is classified as a defeater (not a generic "unless") **only** if the referenced node's metadata contains `defeats_node == current_node.id`. This is a bidirectional pointer check.
5. **No mutation.** The function reads `network.nodes`, `network.nogoods`, `network.meta` but never writes. The `Network` object is untouched.
6. **Beliefs type fallback.** If `beliefs_type` metadata is empty/absent, the type is inferred: `DERIVED` if the node has at least one justification, `OBSERVATION` otherwise. This invariant keeps the output conformant to the beliefs.md schema, which requires a type token.

## Error Handling

This module **performs no error handling** of its own:

- No `try/except` blocks.
- No input validation (e.g., it does not check that `network.nodes` is a dict, or that each node has `.text`).
- It relies on `.get()` with defaults for optional metadata fields (`node.metadata.get("stale_reason")`, etc.), so missing keys silently produce empty strings or `None`.
- The one subtle risk: in the defeater check, `network.nodes.get(o)` returns `None` if the referenced node doesn't exist, and the code guards with `if o_node and ...`. However, the `all_deps` collection does **not** guard — it assumes antecedent IDs in `j.antecedents` are valid node IDs. If an antecedent references a deleted node, it will still be emitted as a string in the "Depends on" line (no crash, but a dangling reference in the output).
- `build_meta` is trusted to return a dict with the expected keys (`schema_version`, `project_name`, `updated_at`, `node_count`, `generator`). A `KeyError` there would propagate.

In practice, error handling is pushed to the **caller** (`api.py`) and to the **test suite**, which constructs valid `Network` objects.

## Topics to Explore

- [file] `reasons/metadata.py` — `build_meta` is the single source of frontmatter fields; understanding its contract (especially `schema_version` and `generator` strings) is essential before modifying the export format.
- [file] `reasons/import_beliefs.py` — The inverse operation. Studying the import parser reveals exactly which Markdown lines are parseable, which validates (or invalidates) the lines this exporter produces.
- [function] `reasons/network.py:Network` — The `Node` and `Justification` dataclasses define every attribute this exporter reads (`truth_value`, `justifications`, `outlist`, `metadata`, `source_hash`, etc.). The export is only as robust as the invariants those classes enforce.
- [file] `tests/test_roundtrip.py` — Exercises the import → export → import cycle. The assertions here define the *actual* schema contract more authoritatively than the docstring does.
- [general] `defeater-encoding` — The bidirectional `defeats_node` pointer and `defeater_type`/`defeat_reason_type` metadata is a convention shared across `contradictions.py`, `repair.py`, and this exporter. Understanding how defeaters are created and linked will prevent encoding bugs.

## Beliefs

- `export-markdown-preserves-graph-as-lists` — The export encodes directed justification edges as per-node "Depends on" flat lists rather than as explicit edge objects, meaning the Markdown schema is a lossless but structurally different representation of the Network graph.
- `out-node-maps-to-stale-iff-reason-present` — An `OUT` node is rendered with status `STALE` (not `OUT`) if and only if its metadata contains a non-empty `stale_reason` or `retract_reason`; a bare `OUT` node retains the `OUT` label.
- `defeater-classification-requires-bidirectional-pointer` — An outlist entry is emitted as "Defeated by" only when the *referenced* node's metadata carries `defeats_node == <current node id>`; a one-sided outlist entry without this pointer is emitted as a generic "Unless" line.
- `export-markdown-is-pure` — `export_markdown()` performs no I/O, raises no domain-specific exceptions, and leaves the input `Network` object unmutated; the caller owns file writing and error handling.
- `reasons-lib-mirrors-reasons` — `reasons_lib/export_markdown.py` is a byte-identical copy of `reasons/export_markdown.py`, meaning any bug fix or feature addition must be applied to both files (or the divergence will be caught by `tests/test_export_markdown.py` if it imports from `reasons`).
