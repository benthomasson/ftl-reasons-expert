---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Investigate how `--visible-to`, access tags, and `Access denied` behavior are implemented across commands.

# How `--visible-to`, access tags, and `Access denied` work

The implementation is split across the CLI (`reasons/cli.py`), API/domain layer (`reasons/api.py`), and database backend (`reasons/pg.py`). The `reasons_lib/cli.py` and `reasons_lib/api.py` files are only re-export shims, so the real behavior is in `reasons/`.

## 1. CLI surface for `--visible-to`

`reasons/cli.py` registers `--visible-to` on many subcommands rather than in one global handler. Examples:

- `reasons/cli.py:3183`, `3188`, `3215` — read/show-oriented commands with help text like “Only show nodes whose access_tags are a subset…”
- `reasons/cli.py:3268` — a command with “Only allow if access_tags are a subset…”
- `reasons/cli.py:3274` — premises filtering
- `reasons/cli.py:3475`, `3495`, `3501`, `3511`, `3711` — publish/export filtering
- `reasons/cli.py:3553`, `3566`, `3581`, `3642`, `3651`, `3659`, `3685`, `3736`, `3773`, `3896` — additional list/include/show/export commands

The argument is declared with `metavar="TAG,TAG"`, but the exact parser action is not visible from the observations. Tests show single-tag usage such as `--visible-to hr` and `--visible-to public` in `tests/test_cli.py:205`, `240`, `287`, `313`, and `1132`.

At the CLI level there is also separate access-tag plumbing around `reasons/cli.py:43-44`, where `access_tags` is read from `args` and conditionally processed. That appears to be for specifying node tags, separate from `--visible-to`, which controls the caller’s visible tag set.

## 2. Core visibility rule

The central rule is documented and implemented near `reasons/api.py:39-42`:

- A node is visible if **all** of its `access_tags` are contained in `visible_to`.
- Nodes with no `access_tags` are always visible.
- `visible_to=None` means no filtering.

The code reads tags from node metadata:

```python
tags = node.metadata.get("access_tags", [])
```

This means access tags are not a separate database field in the observed API surface; they are stored inside node metadata.

The tests in `tests/test_access_tags.py` confirm the semantics:

- `test_list_nodes_visible_to_none_shows_all` — no `visible_to` shows all.
- `test_list_nodes_untagged_always_visible` — untagged nodes remain visible.
- `test_list_nodes_subset_match` — `visible_to=["finance", "hr"]` can see a node tagged `["finance"]`.
- `test_list_nodes_multi_tag_requires_all` — a node tagged `["finance", "hr"]` is hidden from `visible_to=["finance"]` and visible only when both tags are present.

## 3. Storing and inheriting access tags

`add_node` accepts an `access_tags` parameter at `reasons/api.py:288`, documents it at `reasons/api.py:307`, passes it through at `reasons/api.py:318`, and stores it in metadata at `reasons/api.py:377-378`:

```python
if access_tags:
    metadata["access_tags"] = sorted(set(access_tags))
```

So tags are deduplicated and sorted when stored.

Update/rewrite paths preserve existing tags using patterns like `reasons/api.py:690-691` and `reasons/api.py:1205-1206`:

```python
old_tags = old_node.metadata.get("access_tags")
metadata = {"access_tags": old_tags} if old_tags else {}
```

Tag inheritance is tested heavily in `tests/test_access_tags.py`:

- A derived node inherits a single parent’s tags.
- A derived node inherits the union of multiple parents’ tags.
- Explicit tags are merged with inherited tags.
- Premises do not inherit tags.
- Chain and diamond inheritance propagate correctly.
- Adding a justification updates a node’s tags and propagates them to dependents.
- Tags persist through storage.

This inheritance is security-relevant: `test_explain_inherits_restriction_from_antecedent` shows that if a derived node depends on a restricted antecedent, the derived node inherits the restriction and cannot be explained by a caller who cannot see the antecedent.

## 4. Filtering vs hard permission failures

There are two broad behaviors across commands.

### Filtering commands

APIs like `list_nodes`, `lookup`, `search`, and `get_status` accept `visible_to` and filter results instead of raising. The observations show:

- `reasons/api.py:2929` — `visible_to` documented for a list-like API.
- `reasons/api.py:3038` — `visible_to` documented for another search/list-like API.
- `reasons/api.py:4113-4114` and `4279-4280` — result filtering checks:

```python
if not v.get("metadata", {}).get("access_tags")
or all(t in tags for t in v["metadata"]["access_tags"])
```

Tests in `tests/test_access_tags.py` confirm filtering behavior for:

- `list_nodes`
- `lookup`
- `search`
- `get_status`
- search neighbor expansion

For example, `test_search_neighbors_filtered` verifies that restricted neighbors are not exposed through search expansion.

### Direct-read commands

Commands that directly read a specific node—`show`, `explain`, `trace`, and `trace-access-tags`—raise `PermissionError` when the target node is not visible.

The observations show docstrings for these permission checks at:

- `reasons/api.py:875` — `show_node`
- `reasons/api.py:1037` — `explain_node`
- `reasons/api.py:1059` — `trace_assumptions`
- `reasons/api.py:1085` — `trace_access_tags`

Tests assert this directly:

- `test_show_node_raises_on_forbidden`
- `test_explain_raises_on_forbidden`
- `test_trace_raises_on_forbidden`

CLI tests then assert that the error is surfaced as `Access denied` for `show`, `explain`, `trace`, and `trace-access-tags` at `tests/test_cli.py:242`, `289`, `315`, and `1134`.

## 5. `trace_access_tags` and backend enforcement

`reasons/api.py:1079-1098` implements `trace_access_tags`. Its documented behavior is:

- Trace backward through dependency chains.
- Return `{"node_id": ..., "access_tags": [...]}`.
- Raise `PermissionError` if the node’s `access_tags` are not a subset of `visible_to`.

When a PostgreSQL connection is configured, `trace_access_tags` dispatches through `_pg_dispatch` at `reasons/api.py:1088`. The PostgreSQL layer explicitly raises:

```python
raise PermissionError(f"Access denied for node '{node_id}'")
```

at `reasons/pg.py:697`.

So there are two enforcement points visible in the observations:

- API-level visibility checks and documented `PermissionError` behavior.
- Database-layer `PermissionError` construction in `reasons/pg.py`.

## 6. How `Access denied` reaches the CLI

The CLI catches permission failures and prints them to stderr at:

- `reasons/cli.py:522`
- `reasons/cli.py:645`
- `reasons/cli.py:817`
- `reasons/cli.py:835`

Each uses:

```python
print(f"Access denied: {e}", file=sys.stderr)
```

Because the PostgreSQL layer raises a message like `Access denied for node 'fin'`, the CLI output can become `Access denied: Access denied for node 'fin'`. Tests only assert the substring `Access denied`, so this duplication is not a problem for the current contract.

The repeated print statements suggest the CLI translates `PermissionError` in multiple command handlers rather than through a single global exception handler.

## 7. What the observations do not show

The following details are not fully visible:

- The exact subcommand names associated with every `--visible-to` declaration in `reasons/cli.py`.
- The argparse action used for `--visible-to` and whether comma splitting, repeated flags, or both are supported.
- The full implementation of the core visibility predicate near `reasons/api.py:39-42`.
- The full PostgreSQL query logic around `reasons/pg.py:697`.
- Exact CLI exit codes after printing `Access denied`.
- Full publish/export behavior beyond the help text and option declarations.

## Topics to Explore

- [file] `reasons/cli.py` — Map every `--visible-to` registration from lines 3183-3896 to its subcommand and confirm parser normalization.
- [function] `reasons/api.py:trace_access_tags` — Inspect the backward dependency traversal, union-of-tags calculation, and permission gate.
- [file] `reasons/api.py` — Audit all APIs for filter-vs-raise behavior and tag inheritance/preservation paths.
- [file] `reasons/pg.py` — Trace callers of the `PermissionError` at line 697 and see how database queries enforce visibility.
- [general] `visible-to-command-matrix` — Build a matrix of commands showing whether `--visible-to` filters results, blocks a direct read, or restricts publish/export output.

## Beliefs

- `visible-to-is-subset-check` — A node with `access_tags` is visible only when every tag is present in `visible_to`; untagged nodes are always visible.
- `access-tags-inherit-through-antecedents` — Derived nodes receive the union of antecedent `access_tags` plus explicit tags, and new justifications propagate tags to dependents.
- `direct-reads-raise-permission-error` — `show`, `explain`, `trace`, and `trace_access_tags` raise `PermissionError` when the target node is not visible.
- `list-search-status-lookup-filter-results` — `list_nodes`, `search`, `lookup`, and `get_status` filter out invisible nodes rather than raising, including search neighbor expansion.
- `cli-translates-permission-error-to-access-denied` — CLI handlers print `Access denied: {e}` to stderr when a permission failure escapes from the API or database layer.
