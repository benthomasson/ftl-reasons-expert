---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: `_is_visible` semantics (`all()` subset check), and where else (search/compact/export) it gates node visibility.

## `_is_visible` — Semantics and Gating Points

### The check itself

Defined at `reasons/api.py:36`:

```python
def _is_visible(node, visible_to: list[str]) -> bool:
    tags = node.metadata.get("access_tags", [])
    if not tags:
        return True
    visible_set = set(visible_to)
    return all(t in visible_set for t in tags)
```

Two things to internalize:

1. **Untagged nodes short-circuit to `True`.** Absence of `access_tags` means "public" — no caller can be denied. Confirmed by `tests/test_access_tags.py:test_list_nodes_untagged_always_visible`.
2. **Tagged nodes use `all(… in …)` — a subset check, not an intersection check.** The node's tag set must be **⊆** the caller's `visible_to`. In other words, the caller must possess *every* tag attached to the node.

Why `all()` and not `any()`: `test_list_nodes_multi_tag_requires_all` shows a node tagged `["finance","hr"]` is **invisible** to a caller with `visible_to=["finance"]`. The tag set is unioned from every ancestor at write time (`reasons/network.py`, exercised by `TestInheritance` in the same test file — e.g. `test_derived_inherits_union_of_parents`), so a derived node carries the union of all upstream provenance. Using `any()` here would let a caller with `finance` see a claim that also depends on `hr` data — i.e. it would leak. The subset rule is the only safe default once inheritance unions tags.

### `visible_to is None` disables the filter

Every reference site outside the definition is guarded by `visible_to is not None` (or inverted: `visible_to is None or _is_visible(...)`). Passing `None` means "no ACL applied" — it does *not* mean "empty tag list." This is why `test_list_nodes_visible_to_none_shows_all` and `test_show_node_no_filter` both pass.

### Where it gates node visibility

All references are in `reasons/api.py` except a parallel method in `reasons/pg.py`. Grouped by idiom:

**A. Hard denials — raise `PermissionError` (point lookups).**
Lines around 1046, 1069, 1093 (`show_node`, `explain_node`, `trace_assumptions` family) use `if visible_to is not None and not _is_visible(net.nodes[node_id], visible_to): …`. Tests: `test_show_node_raises_on_forbidden`, `test_explain_raises_on_forbidden`, `test_trace_raises_on_forbidden`, and `test_explain_inherits_restriction_from_antecedent` (a derived node inherits the parent tag, so the derived node also raises for an under-privileged caller).

**B. Premise filtering during trace.**
`reasons/api.py:1075` — `premises = [p for p in premises if p in net.nodes and _is_visible(net.nodes[p], visible_to)]`. This is the `trace_assumptions` premise filter asserted by `test_trace_filters_premises`.

**C. Positive inclusion in exports (`visible_to is None or _is_visible(...)`).**
`reasons/api.py:1869` builds a filtered ID list that becomes `data["nodes"]`. Backs `test_export_network_respects_visible_to`.

**D. Nogood gating in `export_network`.**
`reasons/api.py:1882`:

```python
if visible_to is None or all(n in net.nodes and _is_visible(net.nodes[n], visible_to) for n in ng.nodes)
```

A nogood is kept only if **every** member node is visible. `test_export_network_filters_nogoods` proves this: `["pub","fin"]` nogood vanishes under `visible_to=["public"]` and reappears under `["finance","public"]`. The `n in net.nodes` guard also silently drops nogoods that reference deleted nodes.

**E. Per-candidate filters in `search` / `lookup`.**
References at 2601, 2671 (bare `if _is_visible(node, visible_to):` — the enclosing function guarantees `visible_to is not None`), plus filter-and-`continue` patterns at 2911/2952. Behavioural evidence: `test_search_respects_visible_to`, `test_lookup_respects_visible_to`, and critically `test_search_neighbors_filtered` — a public node derived from a hidden `secret` node is shown by search, but the `secret` antecedent is **not** expanded into the result. So `_is_visible` gates not only the matched node but also the antecedent/neighbor expansion loop.

**F. Node-list filtering before `compact` / `export_markdown`.**
Neither lower-level module knows about `visible_to`:

- `reasons/compact.py:compact(network, budget, truncate)` iterates `network.nodes.values()` and splits into `in_nodes`/`out_nodes` — no filter parameter.
- `reasons/export_markdown.py:export_markdown(network, repos)` iterates `network.nodes.values()` (the `sorted(...)` line) — same.

So gating must happen in `reasons/api.py` before these are invoked. That's almost certainly what the `if nid in net.nodes and _is_visible(net.nodes[nid], visible_to)` comprehensions at **3093** and **3156** are doing — building a filtered node set / ID list passed down. Backed by `test_compact_respects_visible_to` and `test_export_markdown_respects_visible_to`.

**G. Status listing and other bulk views.**
Lines 814, 885, 3465, 3648, 3982, 4961 all use the `if visible_to is not None and not _is_visible(...): continue/skip` idiom. `test_status_respects_visible_to` exercises the `get_status` path.

**H. PostgreSQL backend.**
`reasons/pg.py` has its own `self._is_visible(meta, visible_to)` at 666, 696, 775, 986, 1102, 1309, 1361, 1849, 2157, 3446, 3581. Note the first argument is `meta`, not `node` — it takes a metadata mapping directly because the PG backend works over rows rather than `Network` node objects. The guard pattern is identical (`if visible_to is not None and not self._is_visible(meta, visible_to):`).

### What the observations do not show

- The exact enclosing function for each `api.py` line number. The observation dump lists line numbers and code fragments, but not function boundaries — I mapped categories to tests, not to definitively-named functions for every line.
- The body of `PgApi._is_visible` (only call sites are listed; the definition itself is not in the observations).
- The full bodies of `search`, `lookup`, `compact`, `export_network`, `export_markdown`, `list_nodes`, `show_node`, `explain_node`, `trace_assumptions`, `get_status` — only excerpts and tests.
- Whether `export_network` builds a filtered `Network` clone or filters dict output in place. The test (`"pub" in data["nodes"]`) implies dict-level filtering of the return value; the mechanism is not shown.

## Topics to Explore

- [function] `reasons/pg.py:_is_visible` — PostgreSQL variant takes `meta` instead of a node object; compare the subset semantics and confirm tag-loading path parity with the sqlite/Network path.
- [function] `reasons/api.py:export_network` — Confirms how the `visible_to is None or _is_visible(...)` inclusion list (line 1869) and nogood filter (line 1882) compose before the return dict is built.
- [function] `reasons/api.py:search` — Read the antecedent/neighbor expansion loop to see how `_is_visible` is applied per-neighbor, and where the `visible_to is not None` guard is set once for the whole call.
- [function] `reasons/api.py:compact` — Confirms that the API layer builds a filtered node set at lines ~3093/3156 before delegating to `reasons/compact.py:compact`, which itself has no `visible_to` parameter.
- [general] `visibility-gating-patterns` — Catalog whether each public API either **raises** `PermissionError` (point lookups) or **filters silently** (bulk views/exports), since that behavioral distinction is a caller-facing contract.

## Beliefs

- `_is-visible-is-subset-check` — `_is_visible` returns `True` iff the node's `access_tags` set is a subset of `visible_to` (via `all(t in visible_set for t in tags)`), or the node has no tags.
- `untagged-nodes-are-always-visible` — A node with empty/missing `access_tags` short-circuits to `True` before any set comparison, so untagged nodes cannot be hidden by `visible_to`.
- `visible-to-none-disables-filtering` — Every `_is_visible` call site is guarded so `visible_to=None` bypasses the check entirely; `None` means "no ACL" rather than "empty tag list."
- `export-network-requires-all-nogood-nodes-visible` — `api.export_network` omits a nogood unless **every** member node passes `_is_visible` (line 1882), so partially-visible nogoods are dropped entirely.
- `compact-and-export-markdown-lack-visible-to-parameter` — `reasons/compact.py:compact` and `reasons/export_markdown.py:export_markdown` iterate `network.nodes.values()` without any visibility parameter, so gating must be applied in `reasons/api.py` before delegation.
