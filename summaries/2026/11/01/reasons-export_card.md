---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/export_card.py

# `reasons/export_card.py` — Explanation

## 1. Purpose

This module owns the **serialization of a reasons belief network into a HuggingFace-compatible "EEM card"** — a single `README.md` string with YAML frontmatter and a markdown body. It is the export path that makes a network portable and shareable on the HuggingFace hub. The module does not write to disk; it returns a string, leaving I/O to the caller (e.g., `reasons/api.py`).

Think of it as the "card generator" — analogous to how `hub.create_repo` produces a `README.md`, but tailored to the EEM (Epistemic Evidence Model) schema.

## 2. Key Components

### `_node_depth(nid, nodes, memo=None) → int`

A private recursive helper that computes the **maximum derivation depth** from a leaf (justification-free node) up to the given node, following only the *supporting* justification when one is marked.

| Parameter | Type | Contract |
|-----------|------|----------|
| `nid` | `str` | A valid node ID present in `nodes` (or absent, in which case it returns 0) |
| `nodes` | `dict[str, Node]` | The full node map from `Network.nodes` |
| `memo` | `dict[str, int] \| None` | Shared memo across the traversal; caller passes the same dict for all top-level calls |

**Cycle guard:** Before recursing into antecedents, it sets `memo[nid] = 0`. If a justification cycle ever exists (it shouldn't in a well-formed network), this prevents infinite recursion — the back-edge simply resolves to depth 0.

**Supporting-justification optimization:** If `node.supporting_justification` is a valid index, only that single justification's antecedents are traversed. Otherwise *all* justifications are considered. This means the reported depth reflects the "live" proof path, not the full justification history.

### `export_card(network, domain, license, base_network, source_repos) → str`

The public entry point. Its contract:

- **Input:** A fully-constructed `Network` object plus optional card metadata.
- **Output:** A complete `README.md` string (YAML frontmatter + markdown body), newline-joined.
- **Side effects:** None. Pure function over the network's current state.
- **Key stats computed internally:**
  - `in_count` / `out_count` — partition of nodes by `truth_value == "IN"`
  - `premises` — nodes with no justifications (observations)
  - `derived` — complement of premises
  - `nogood_count` — `len(network.nogoods)`
  - `max_depth` — via `_node_depth` over all nodes
  - `retraction_pct` — `out_count / total * 100`, zero-guarded

## 3. Patterns

- **Memoized DFS for DAG depth.** Classic tree-depth with memoization; the shared `memo` dict is created once in `export_card` and threaded through all top-level calls to amortize subgraph revisits.
- **String-building via list-of-lines.** The output is assembled as `lines: list[str]` and joined with `"\n".join(lines)` at the end. This is the idiomatic Python way to build large multi-line text without f-string nesting.
- **Frontmatter + body separation.** The YAML block (bounded by `---`) is appended first, then a blank line, then the markdown. This mirrors the HuggingFace card convention where parsers expect frontmatter at byte 0.
- **Defensive index bounds check.** The `supporting_justification` lookup is guarded: `0 <= si < len(node.justifications)`. If the index is stale (e.g., after a justification was removed), it falls back to considering all justifications rather than crashing.

## 4. Dependencies

### Imports (this file depends on)

| Module | What it uses | Why |
|--------|-------------|-----|
| `reasons.metadata:build_meta` | Returns a dict with `project_name`, `schema_version`, `generator` | Standardizes the metadata block so it matches what other tools (import, verify) expect |
| `reasons.network:Network` | Type annotation only | The function signature documents the expected shape; no runtime import cost beyond the annotation |

### Imported By (depends on this file)

| Consumer | Usage |
|----------|-------|
| `reasons/api.py` | The high-level API likely exposes `export_card` as part of the public surface (e.g., `client.export_card()`) |
| `reasons_lib/export_card.py` | Mirror copy for the distributable library package |
| `tests/test_export_card.py` | Unit tests verifying the generated string structure |

## 5. Flow

```
export_card(network, ...)
│
├─ 1. build_meta(project_name, node_count, created_at)
│       → { project_name, schema_version, generator }
│
├─ 2. Compute stats
│     total, in_count, out_count, premises, derived, nogood_count
│
├─ 3. Compute max_depth
│     memo = {}
│     for each node_id: _node_depth(node_id, nodes, memo)
│     → max across all nodes
│
├─ 4. Compute retraction_pct (zero-guarded)
│
├─ 5. Build YAML frontmatter (lines 0..N)
│     schema_version, type, project_name, domain, license,
│     base_network, source_repos, belief counts, generator
│
├─ 6. Build Markdown body
│     # Title
│     ## Stats (table)
│     ## How to Use (bash snippets)
│     ## Files (table)
│     ## Quality (bullets)
│     ## Limitations
│     ## License
│
└─ 7. return "\n".join(lines)
```

The entire function is **synchronous, allocation-heavy, and I/O-free**. A 10k-node network produces a string in roughly O(V × J) time where J is average justifications-per-node (dominated by the depth computation).

## 6. Invariants

- **Frontmatter always first.** The output string always begins with `---\n` and contains exactly one `---` delimiter pair before the markdown body. HuggingFace parsers depend on this.
- **`type: eem` is mandatory and unconditional.** It is the discriminator that tells the hub this is an EEM, not a generic model card.
- **`in_count + out_count == total`.** The partition is exhaustive: every node is either `"IN"` or not. There is no third state.
- **`premises + derived == total`.** A node is a premise iff it has zero justifications; otherwise it is derived.
- **Depth is ≥ 0 for all nodes.** Leaves (no justifications) get 0; everything else gets at least 1.
- **Retraction rate is a whole number.** The `:.0f` format specifier rounds to the nearest integer percentage.

## 7. Error Handling

This module is deliberately **error-minimal**:

| Scenario | Handling |
|----------|----------|
| `network.nodes` is empty | `max(..., default=0)` prevents a `ValueError`; `retraction_pct` is guarded by `if total > 0` |
| `node.supporting_justification` is `None` | Falls through to the `else` branch (all justifications) |
| `node.supporting_justification` is out of range | Bounds check `0 <= si < len(...)` catches it; falls through to `else` |
| `nid` not in `nodes` dict | `nodes.get(nid)` returns `None`; the `if not node` guard returns 0 |
| Cyclic justification graph (shouldn't happen) | `memo[nid] = 0` before recursion acts as a visited-marker; back-edge resolves to 0, preventing infinite loop |
| `network.meta` missing keys | `.get("project_name", "")` and `.get("created_at", "")` return safe defaults |

**No exceptions are raised by this module under any documented input.** It is a pure string formatter — if the network is malformed, the depth calculation silently under-reports rather than crashing. A developer maintaining this should be aware that a truly broken network (e.g., antecedent IDs pointing to non-existent nodes) will not produce an error here; it will just produce a lower depth number.

---

## Topics to Explore

- [file] `reasons/metadata.py` — `build_meta` is the only external logic this module calls; understanding its schema guarantees and versioning policy is essential before changing the frontmatter format
- [file] `reasons/network.py` — The `Network` class and `Node` model (especially `supporting_justification`, `truth_value`, `justifications`) define the data contract this exporter reads
- [function] `reasons/export_card.py:export_card` — Compare with `reasons/export_markdown.py` to understand the design distinction between "card" (hub-facing) and "markdown" (human-facing) exports
- [file] `tests/test_export_card.py` — The test suite reveals which frontmatter fields are contractually required by downstream parsers (HuggingFace hub, `reasons import-json`)
- [general] `eem-schema-versioning` — The `schema_version` in frontmatter implies a versioned protocol; explore `reasons/metadata.py` and `CHANGELOG.md` to understand how breaking changes to the card format are signaled

---

## Beliefs

- `export-card-produces-self-contained-readme` — `export_card` returns a complete `README.md` string that requires no external file references to be valid on the HuggingFace hub
- `node-depth-uses-supporting-justification-only` — When `supporting_justification` is a valid index, `_node_depth` traverses only that one justification's antecedents, ignoring sibling justifications
- `export-card-is-pure` — The function has no I/O, no network calls, and no mutable shared state; it can be called concurrently with different networks without synchronization
- `frontmatter-type-is-fixed-eem` — The YAML field `type: eem` is hardcoded and unconditional; removing or changing it breaks HuggingFace hub categorization
- `retraction-pct-zero-guards-empty-network` — When `network.nodes` is empty, the retraction rate is the literal string `"0%"`, not a division-by-zero exception
