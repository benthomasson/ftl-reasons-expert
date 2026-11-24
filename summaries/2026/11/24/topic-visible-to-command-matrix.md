---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Build a matrix of commands showing whether `--visible-to` filters results, blocks a direct read, or restricts publish/export output.

# `--visible-to` behavior matrix

## Important framing correction

The observations do **not** show a `--visible-to` CLI flag. The `find_cli_argparse` grep (`add_argument\(.*visible`) returned **0 matches**, and no `argparse` file appears in the observations. What the code actually exposes is a Python-API keyword argument:

```python
visible_to: list[str] | None = None
```

at `reasons/api.py:36` (`_is_visible`), `reasons/api.py:782` (`get_status`), `reasons/api.py:870` (`show_node`), `reasons/api.py:1032` (`explain_node`), `reasons/api.py:1054` (`trace_assumptions`), `reasons/api.py:1079` (`trace_access_tags`), and `reasons/api.py:1814` (`export_network`). The matrix below is built on those API functions; if there is a CLI wrapper, it lives outside the scope of these observations.

## The visibility rule (from `_is_visible`, `reasons/api.py:36-47`)

A node is visible iff **every** entry in its `metadata["access_tags"]` is in `visible_to` — subset semantics, not "any overlap". Nodes with no `access_tags` are always visible. `visible_to=None` disables the filter entirely.

Two consequences pinned by tests:
- A node tagged `["finance", "hr"]` is **not** visible to `["finance"]` alone (`test_list_nodes_multi_tag_requires_all`, `tests/test_access_tags.py`).
- A derived node inherits the **union** of its antecedents' tags (`test_derived_inherits_union_of_parents`), so an unreachable ancestor transitively blocks the descendant.

## Matrix: role each command plays

| Command (API function) | Filters results | Blocks a direct read | Restricts publish/export output | Evidence |
|---|---|---|---|---|
| `list_nodes` | ✅ silently omits | – | – | `test_list_nodes_filters_by_access`, `test_list_nodes_subset_match` |
| `lookup` | ✅ | – | – | `test_lookup_respects_visible_to` |
| `search` | ✅ (also neighbors) | – | – | `test_search_respects_visible_to`, `test_search_neighbors_filtered` |
| `get_status` | ✅ | – | – | `reasons/api.py:814`, `test_status_respects_visible_to` |
| `compact` | ✅ | – | – | `test_compact_respects_visible_to` |
| `show_node` | – | ✅ `PermissionError` | – | `reasons/api.py:885-887`, `test_show_node_raises_on_forbidden` |
| `explain_node` | – | ✅ `PermissionError` | – | `reasons/api.py:1046-1048`, `test_explain_raises_on_forbidden` |
| `trace_access_tags` | – | ✅ `PermissionError` | – | `reasons/api.py:1093-1095`, `test_api_trace_access_tags_raises_on_forbidden` |
| `trace_assumptions` | ✅ premises list (secondary) | ✅ target node | – | `reasons/api.py:1069-1075`, `test_trace_filters_premises` |
| `export_network` | ✅ nodes + nogoods | – | ✅ (this is the publish artifact) | `reasons/api.py:1814`, `test_export_network_respects_visible_to`, `test_export_network_filters_nogoods` |
| `export_markdown` | ✅ | – | ✅ | `test_export_markdown_respects_visible_to` |
| `pubsub` / live publish | – | – | Not gated (see caveat) | `grep_export_publish_vis` returned 0; scope unclear |

### The three roles explained

1. **Filter-only commands** (`list_nodes`, `lookup`, `search`, `get_status`, `compact`): when a node's tags aren't a subset of `visible_to`, it is quietly dropped from the result set — no error is raised. `search` additionally filters the neighbor/expansion set (`test_search_neighbors_filtered`), so secrets don't come in through the back door of a graph walk.

2. **Block-only commands** (`show_node`, `explain_node`, `trace_access_tags`): when the target node fails `_is_visible`, these raise `PermissionError` with a message naming the node and `visible_to` (`reasons/api.py:887`, `1048`, `1095`). They never return partially-redacted data for an unreadable node.

3. **Block-then-filter** (`trace_assumptions`, `reasons/api.py:1069-1075`): first the target must pass `_is_visible` (raises otherwise); then the returned `premises` list is filtered again with `_is_visible`, so an accessible node doesn't leak inaccessible premise IDs.

4. **Publish/export restriction** (`export_network` at `reasons/api.py:1814`, `export_markdown`): the artifact itself is filtered — the exported `nodes` dict and `nogoods` list only contain what the caller is allowed to see. `test_export_network_filters_nogoods` proves nogoods that reference any invisible node are dropped too.

### Why `explain_node` and `trace_assumptions` don't leak antecedents

This is the subtle part. Those two functions only check `_is_visible` on the **target** node, not on each antecedent. Safety comes from **tag inheritance**: a derived node carries the union of its antecedents' tags (`tests/test_access_tags.py:TestInheritance`). So `explain_node("derived", visible_to=["public"])` where `derived` was derived from a `finance`-tagged parent fails at the top-level check (`test_explain_inherits_restriction_from_antecedent`), because `derived` itself inherited `["finance"]`. Same guard for `trace_assumptions`.

## Caveats / what's missing in the observations

- **No CLI flag confirmed.** `find_cli_argparse` returned 0 matches. If `--visible-to` exists, it wasn't captured — search the CLI entrypoint (`reasons/cli.py` or similar) before asserting anything about a shell flag.
- **Two greps return 0 despite contradictory evidence.** `grep_visible_to_in_tests` and `grep_export_publish_vis` both return 0, yet `tests/test_access_tags.py` clearly uses `visible_to`, and `reasons/api.py:1814` uses it too. Those greps were evidently run against a narrower scope (probably the `pubsub`/publish module only). For **publish**, this is the only signal we have and it is ambiguous; treat "publish is not gated" as an open question, not a finding.
- **Write paths are not covered.** `add_node`, `retract_node`, `update_*`, etc. take `access_tags` but no `visible_to`, so this matrix only covers reads and exports.
- **`get_status`'s exact harvest path is not visible** — we only see line 814's filter and the counter update at line 801. Confirm whether `total` in the result is the post-filter count (the test implies yes: `assert result["total"] == 1`).
- **Interaction with `include_out`** (from `tests/test_hide_out.py`) is orthogonal; both filters run, but ordering isn't shown.

## Topics to Explore

- [file] `reasons/api.py` — read lines 780-900 and 1020-1100 to see the exact filter-vs-raise branches side by side
- [function] `reasons/api.py:export_network` — verify how both `nodes` and `nogoods` are filtered around line 1814 (the nogoods guard isn't shown in the snippet)
- [file] `tests/test_access_tags.py` — the canonical behavior spec; every matrix row above has a test
- [function] `reasons/network.py:trace_access_tags` — where the ancestor-tag union (the safety net for explain/trace) is actually computed
- [general] `visible-to-cli-exposure` — confirm whether any CLI entrypoint wires `visible_to` to an argparse flag; observations show none
- [general] `publish-visibility-gap` — determine whether `reasons/pubsub.py` (live publish) is a leakage path that `--visible-to` does not cover

## Beliefs

- `no-cli-visible-to-flag-in-observations` — The only surfaced grep for argparse (`find_cli_argparse`) returned 0 matches, so `visible_to` is exposed as a Python kwarg, not a shell flag, within the captured scope.
- `is-visible-implements-subset-semantics` — `_is_visible` (`reasons/api.py:36-47`) treats a node as visible iff every `access_tags` entry is contained in `visible_to`; untagged nodes are always visible.
- `show-node-blocks-via-PermissionError` — `show_node` raises `PermissionError` at `reasons/api.py:885-887` instead of silently filtering, unlike list/search/lookup/status/compact.
- `tag-inheritance-prevents-explain-leak` — Derived nodes inherit the union of antecedent `access_tags` (`tests/test_access_tags.py:TestInheritance`), which is what makes `explain_node` and `trace_assumptions` safe without per-antecedent visibility checks.
- `trace-assumptions-blocks-target-then-filters-premises` — `trace_assumptions` raises `PermissionError` on the target when invisible (`reasons/api.py:1069-1071`) and additionally filters the returned `premises` (`reasons/api.py:1074-1075`).
- `export-network-filters-nodes-and-nogoods` — `export_network` accepts `visible_to` (`reasons/api.py:1814`) and drops nogoods referencing invisible nodes (`test_export_network_filters_nogoods`), so the exported artifact — not just the query result — is scoped.
