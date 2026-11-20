---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: The subset rule, omitted-filter behavior, and silent-filter vs `PermissionError` split across API methods.

# Access-control semantics in `reasons/api.py`

Here's how the three behaviors fit together.

## 1. The subset rule

The core predicate is `_is_visible` in `reasons/api.py:39-46`:

```python
def _is_visible(node, visible_to: list[str]) -> bool:
    tags = node.metadata.get("access_tags", [])
    if not tags:
        return True
    visible_set = set(visible_to)
    return all(t in visible_set for t in tags)
```

Three invariants follow from this:

- **Untagged means public.** If a node has no `access_tags`, it's always visible regardless of what `visible_to` is (`test_list_nodes_untagged_always_visible`, `tests/test_access_tags.py:127`).
- **Subset, not intersection.** A node is visible only when **every** one of its `access_tags` appears in `visible_to`. `test_list_nodes_multi_tag_requires_all` (`tests/test_access_tags.py:137`) confirms a node tagged `["finance", "hr"]` is hidden when `visible_to=["finance"]` but visible when `visible_to=["finance", "hr"]`.
- **Superset access is fine.** `test_list_nodes_subset_match` (`tests/test_access_tags.py:151`) shows a `["finance"]`-tagged node is still visible when `visible_to=["finance", "hr"]` — the extra tag in the query doesn't exclude anything.

The docstring at `reasons/api.py:39-40` states it directly: *"A node is visible if its access_tags are all contained in visible_to. Nodes with no access_tags are always visible."*

Contrast with `_filter_by_tags` at `reasons/api.py:71-98`. That function does **not** govern access control — it handles user-supplied topic/status/tag filters with AND-of-groups, OR-within-group semantics (`reasons/api.py:79`, `:91-96`). Don't conflate the two: `_is_visible` is the security check; `_filter_by_tags` is the content-query filter.

## 2. Omitted-filter behavior

Passing `visible_to=None` (or omitting the parameter entirely) **disables the access check**. There is no "deny-all" default and no "empty list means deny-all" behavior either, since an empty node-tag list is always visible.

Evidence:

- `test_list_nodes_visible_to_none_shows_all` (`tests/test_access_tags.py:117`): no `visible_to` → both public and finance nodes returned.
- `test_show_node_no_filter` (`tests/test_access_tags.py:186`): `show_node("fin")` with no `visible_to` returns the finance node.
- `test_list_nodes_visible_to_none_shows_all` also makes the point that `visible_to` is *opt-in* — the caller has to declare a restricted view.

So the semantics are *"if you don't say what you can see, you see everything"*, not *"if you don't say what you can see, you see nothing"*. That matches the shape of the callers in `reasons/cli.py` (e.g. `--visible-to` at `cli.py:3183`, `:3188`, …): the flag is optional, and when absent the CLI just omits it.

## 3. Silent filter vs `PermissionError` split

The control flow branches on whether the API method is **enumeration** or **named retrieval**, not on whether access is denied.

### Enumeration methods silently filter

These take a set of candidate node IDs and drop the invisible ones before returning. No exception, no partial-error flag — forbidden nodes are simply absent:

- `list_nodes` — `test_list_nodes_filters_by_access` (`tests/test_access_tags.py:105`)
- `search` — `test_search_respects_visible_to` (`:201`), `test_search_neighbors_filtered` (`:209`)
- `lookup` — `test_lookup_respects_visible_to` (`:193`)
- `get_status` — `test_status_respects_visible_to` (`:216`)
- `export_network` — `:239`, plus nogood filtering at `:248`
- `export_markdown` — `:254`
- `compact` — `:262`
- `list_negative` — the `visible_to` filter is applied to candidates *before* the LLM prompt is built; `test_visible_to` (`tests/test_api.py:388`) asserts the restricted text never appears in the prompt.

The `except (KeyError, PermissionError):` blocks at `reasons/api.py:3596`, `:3714`, `:3763`, `:3834` suggest some enumeration paths further **normalize** a raised error back into a silent skip — e.g. when a downstream helper touches a node the caller can't see, it's treated as "not present" rather than a hard failure.

### Named-retrieval methods raise `PermissionError`

These retrieve **one specific node by ID**. Returning "not found" and "you don't have access" would be indistinguishable to the caller, so instead they raise:

| Method | `PermissionError` raise site | Test |
|---|---|---|
| `show_node` | `reasons/api.py:886` (docstring at `:875`) | `test_show_node_raises_on_forbidden` (`tests/test_access_tags.py:169`) |
| `explain_node` | `reasons/api.py:1047` (docstring at `:1037`) | `test_explain_raises_on_forbidden` (`:228`) |
| `trace_assumptions` | `reasons/api.py:1070` (docstring at `:1059`) | `test_trace_raises_on_forbidden` (`:251`) |
| `trace_access_tags` | `reasons/api.py:1094` (def at `:1079`, docstring at `:1085`) | `test_api_trace_access_tags_raises_on_forbidden` (`:349`) |

The split is intentional and observable from the CLI: `reasons/cli.py` has `except PermissionError as e:` handlers at `:521`, `:644`, `:746`, `:816`, `:834`, so the "permission denied" message can be turned into a user-facing error. Enumeration commands don't need that handler because the API already filtered silently.

### Why `explain_node` must raise

`explain_node` reasons over *antecedents*. Because derived nodes inherit their parents' `access_tags` (see `test_derived_inherits_union_of_parents` at `tests/test_access_tags.py:40` and `test_chain_inheritance` at `:72`), a derived node is at least as restricted as its most restricted parent. `test_explain_inherits_restriction_from_antecedent` (`tests/test_access_tags.py:233`) verifies the consequence: a caller without `finance` can't `explain_node` a finance-derived node, because otherwise the explain body would leak the restricted antecedent text. Raising is the only safe option — a silent "not visible" return would hide the fact that the node exists at all.

The same logic applies to `trace_assumptions` (`test_trace_filters_premises`, `:262`) and `trace_access_tags`.

## What's missing from the observations

- The bodies of `show_node` and `explain_node` aren't in the returned 600-line slice; only the docstrings and `raise` lines are visible. Confirming that `_is_visible` (not `_filter_by_tags`) is the guard used inside each is inferred from the docstring wording *"Raises PermissionError if node's access_tags are not a subset of visible_to"* (`:875`, `:1037`, `:1059`, `:1085`).
- The exact bodies of the four `except (KeyError, PermissionError):` sites (`api.py:3596, 3714, 3763, 3834`) — I can't see which methods catch these, only that they do.
- Whether `reasons/pg.py` (the Postgres mirror) shares the same subset check — the `PermissionError` at `pg.py:697` and `:2158` suggest it does, but the predicate isn't visible here.

## Topics to Explore

- [function] `reasons/api.py:_is_visible` — The single subset-rule predicate; every silent filter routes through it
- [function] `reasons/api.py:explain_node` — Verify that its `PermissionError` guard uses `_is_visible` and that it checks the *target* node before walking antecedents
- [file] `reasons/pg.py` — Mirror implementation; confirm `PgApi` uses the same subset rule and `PermissionError` split (see `pg.py:697`, `:2158`)
- [general] `permission-error-to-cli-message` — Trace how `except PermissionError as e:` handlers at `cli.py:521, 644, 746, 816, 834` format the denial for the user
- [file] `tests/test_access_tags.py` — The behavioral contract; every access-control rule above is encoded as a test

## Beliefs

- `subset-rule-implemented-in-_is_visible` — `_is_visible` at `reasons/api.py:39-46` returns True iff every tag in `node.metadata["access_tags"]` is in `visible_to`, with an empty tag list short-circuiting to True.
- `omitted-visible-to-disables-access-check` — When `visible_to` is `None` or omitted, no access filter is applied and all nodes are returned (see `test_list_nodes_visible_to_none_shows_all`, `test_show_node_no_filter`).
- `list-operations-silently-filter-forbidden-nodes` — `list_nodes`, `search`, `lookup`, `get_status`, `export_network`, `export_markdown`, `compact`, and `list_negative` return only visible nodes and never raise on access denial.
- `named-retrieval-raises-PermissionError` — `show_node` (`api.py:886`), `explain_node` (`:1047`), `trace_assumptions` (`:1070`), and `trace_access_tags` (`:1094`) raise `PermissionError` when the target node fails the subset check.
- `derived-nodes-inherit-antecedent-tags` — Derived nodes carry the union of their antecedents' `access_tags`, so `explain_node`'s `PermissionError` prevents leaking restricted antecedents via the explanation text.
- `cli-converts-PermissionError-to-user-message` — CLI entry points catch `PermissionError` (e.g. `cli.py:521`, `:644`, `:746`, `:816`, `:834`) and surface it as an error string, whereas successful enumeration flows see only pre-filtered results.
