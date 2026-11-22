---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: _format_markdown in reasons/api.py

# `_format_markdown` — Explanation

## Purpose

`_format_markdown` is the default renderer for `search()` results in `reasons/api.py`. It takes the matched node IDs from a search and the BFS-expanded neighbor IDs from the dependency graph, and produces a single human-readable markdown string — one `###` section per matched belief (with status, text, source, dependencies, dependents) followed by a compact bullet list of "related nodes." It exists so the CLI / LangGraph tool layer can print search results directly without the caller having to walk the network object.

## Contract

- **Preconditions**
  - `net` is a `Network` object with a `.nodes: dict[str, Node]` mapping.
  - Every id in `matched_ids` and `neighbor_ids` exists in `net.nodes`; the function does **no** membership checking and will raise `KeyError` otherwise.
  - Each referenced node exposes `.truth_value`, `.text`, `.source`, `.justifications` (each with `.antecedents: list[str]`), and `.dependents` (a sortable iterable of IDs).
- **Postconditions**
  - Returns a markdown string. Matched sections appear in the exact order given by `matched_ids`; the related-nodes bullet list is sorted alphabetically.
  - Each matched section ends with a blank line; the whole document is `"\n".join(parts)`, so section boundaries are single blank lines.
- **Invariants**
  - The function is **pure with respect to the network** — reads only, no mutation, no I/O.
  - The `matched_ids` vs `neighbor_ids` split is semantic only: matched nodes get full detail, neighbors get a one-line stub.

## Parameters

- `net` — a `Network` (see `reasons/network.py`). Must be fully loaded; `net.nodes` is indexed directly.
- `matched_ids: list[str]` — ordered IDs of primary search hits. Order is **preserved** (the caller in `search()` has already applied relevance/date sorting and pagination).
- `neighbor_ids: set[str]` — related nodes discovered via BFS up to `depth` hops. Order is **not** preserved here (it is a set); the function sorts it before output. Neighbors may be empty.

## Return Value

A single string. Two shapes:

1. **With hits:** one `### <id>` block per match, then (if any neighbors exist) a `---` separator and a `**Related nodes:**` bullet list.
2. **Empty inputs:** `""` — but note that `search()` short-circuits to `"No results found."` long before reaching this function, so an empty string is only returned if `_format_markdown` is called directly.

There is no error return path; failures surface as exceptions.

## Algorithm

**Matched block loop** (for each `nid` in `matched_ids`, order preserved):

1. Append heading `### {nid}`.
2. Append `**Status:** {node.truth_value}`.
3. Append the raw node text (`node.text`).
4. If `node.source` is truthy, append `**Source:** {node.source}`.
5. If `node.justifications` is non-empty:
   - Collect **all** `j.antecedents` across **all** justifications into one flat `deps` list.
   - If that flat list is non-empty, append `**Depends on:** {', '.join(deps)}`.
   - Note: no dedup, and no way to tell the reader that multiple justifications represent *alternate* (OR) support. A node supported by `[A,B]` **or** `[C]` prints as `Depends on: A, B, C`.
6. If `node.dependents` is non-empty, append `**Depended on by:** {', '.join(sorted(node.dependents))}` (sorted for deterministic output).
7. Append an empty string, which becomes a blank line after the join.

**Neighbor block** (only if `neighbor_ids` is non-empty):

8. Append `---` (horizontal rule).
9. Append the literal string `"**Related nodes:**\n"` — combined with the `"\n"` join this produces an extra blank line after the header for markdown rendering.
10. For each `nid` in `sorted(neighbor_ids)`:
    - Append `- **{nid}** ({truth_value}): {text}`.
11. Append an empty string.

**Assembly:** `return "\n".join(parts)`.

## Side Effects

None. No mutations to `net`, no file/DB/network I/O, no logging. It is safe to call on a cached, read-only `Network` (the cache that `_with_network` maintains).

## Error Handling

- **No explicit exception handling.** A missing ID in either list causes `net.nodes[nid]` to raise `KeyError`.
- If a `Node.dependents` element is non-sortable (e.g. a mixed-type set), `sorted(...)` raises `TypeError`. In practice `dependents` holds strings.
- Silent, lossy behavior: because antecedents are flattened across justifications, the output is **not a faithful model of the TMS semantics** — it can misrepresent OR support as AND-like dependency. The reader must know this.

## Usage Patterns

Called from `search()`:

```python
if format == "json":
    return _format_json(net, matched_ids, neighbor_ids)
elif format == "minimal":
    return _format_minimal(net, matched_ids, neighbor_ids)
elif format == "compact":
    return _format_compact(net, matched_ids, neighbor_ids)
else:
    return _format_markdown(net, matched_ids, neighbor_ids)
```

The caller obligations are:

- **Filter IDs before invoking.** `search()` strips ids that aren't in `net.nodes`, drops OUT nodes unless `include_out=True`, applies namespace/tag/visibility filters, and paginates **before** neighbor expansion — so `_format_markdown` can trust its inputs.
- **Ensure `neighbor_ids` was already de-duped against matches** (`neighbor_ids -= set(matched_ids)` in `search()`), so a node doesn't appear both as a full section and a bullet.
- **Skip neighbor expansion** for the `"names-only"` format; that branch never reaches this function.

Because `search()` is also where FTS/regex/OR search happens (`_fts_search`, `_regex_search`, `_or_search`, `_substring_search`), `_format_markdown` never needs to know *how* the match set was produced.

## Dependencies

- **Internal:** `Network` / `Node` from `reasons/network.py` (via `from .network import Network`); `Justification` from `reasons/__init__.py` (its `.antecedents`, `.outlist`, `.label`, `.type` fields define what can be shown).
- **Standard library:** only builtins (`str.join`, `sorted`). No imports within the function.

---

## Topics to Explore

- [function] `reasons/api.py:search` — The sole caller; shows how matched_ids and neighbor_ids are built, filtered, and formatted, and which branch selects this renderer.
- [function] `reasons/api.py:_format_json` — Alternative structured output that preserves more detail; a good contrast for what markdown loses.
- [file] `reasons/network.py` — Defines `Node`, `Network`, and the `dependents`/`justifications` structures that `_format_markdown` reads directly.
- [file] `tests/test_api.py` — Where expected string output of the search/formatting path is asserted; useful to know exact whitespace expectations.
- [general] `neighbor-expansion-bfs` — The `depth`-driven BFS in `search()` that determines which IDs end up in `neighbor_ids`.

## Beliefs

- `format-markdown-assumes-valid-node-ids` — `_format_markdown` indexes `net.nodes[nid]` with no membership check, so any matched or neighbor ID missing from the network raises `KeyError`.
- `neighbor-ids-are-sorted-and-deduplicated` — Neighbor output is alphabetical because `neighbor_ids` is a set that is `sorted()` at render time, whereas `matched_ids` preserves caller order.
- `depends-on-flattens-justifications` — The `Depends on:` line concatenates antecedents across every justification without dedup or type/label markers, conflating alternate (OR) support with a single dependency list.
- `format-markdown-has-no-side-effects` — The function performs only reads of node fields and string assembly; it never mutates the network or touches I/O.
- `search-defaults-to-format-markdown` — `search()` routes to `_format_markdown` whenever `format` is not `json`, `minimal`, `compact`, or `names-only`, making it the effective default renderer.
