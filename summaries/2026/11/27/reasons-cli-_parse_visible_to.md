---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Function: _parse_visible_to in reasons/cli.py

## Purpose

`_parse_visible_to` is a small CLI-layer adapter that converts the optional `--visible-to` argparse value into the shape expected by `reasons.api` functions: either `None` or a `list[str]`.

It exists because many CLI commands expose `--visible-to TAG,TAG` as a comma-separated string, while API functions generally expect a tag list or `None` for unrestricted visibility. Centralizing the parsing avoids repeating `.split(",")` logic across `cmd_status`, `cmd_show`, `cmd_search`, `cmd_list`, `cmd_publish`, `cmd_compact`, and many others.

## Contract

### Preconditions

- `args` is expected to be an `argparse.Namespace`-like object.
- `args.visible_to` may be missing, `None`, or a string.
- If `args.visible_to` is non-`None`, the function assumes it is a string that supports `.split(",")`.

### Postconditions

- Returns `None` if `visible_to` is missing or explicitly `None`.
- Otherwise returns a `list[str]`.
- Each list item is the stripped form of a comma-separated token.
- Empty or whitespace-only tokens are omitted.
- Order is preserved.
- Duplicates are preserved.
- Tags are not validated, normalized, deduplicated, or sorted.

### Important invariant / edge case

The function uses `if val is not None`, not `if val`.

Therefore:

- Missing attribute or `visible_to=None` → `None`
- `visible_to=""` → `[]`
- `visible_to="   "` → `[]`
- `visible_to=" , , "` → `[]`

This distinction matters because `None` generally means “no filter”, while `[]` may mean “explicitly allow no tags”, depending on how the receiving API interprets it.

## Parameters

### `args`

A namespace-like object containing an optional `visible_to` attribute.

Typical value from argparse:

```python
args.visible_to == "internal,team-a"
```

Expected values:

| Value | Behavior |
|---|---|
| Attribute missing | Returns `None` |
| `None` | Returns `None` |
| `""` | Returns `[]` |
| `"a,b"` | Returns `["a", "b"]` |
| `" a , b , "` | Returns `["a", "b"]` |
| `","` | Returns `[]` |
| `"a,,b"` | Returns `["a", "b"]` |
| `"a a,b"` | Returns `["a a", "b"]` |

The function does not enforce the type. If a caller passes a list, integer, or another non-string object with no `.split()` method, it will raise an `AttributeError`.

## Return Value

Returns one of:

- `None` — no visibility filter / attribute absent
- `list[str]` — non-empty or empty list of parsed tokens

The caller must handle:

- `None` as “do not apply an access-tag filter”
- `list[str]` as the allowed visibility tag set
- `[]` as an explicit parsed value, whose semantics are defined by the receiving API

## Algorithm

Step by step:

1. Read `args.visible_to` using `getattr(args, "visible_to", None)`.
2. If the value is exactly `None`, return `None`.
3. Otherwise, split the string on commas:
   ```python
   val.split(",")
   ```
4. For each token:
   - Strip leading/trailing whitespace with `.strip()`.
   - Keep the stripped token only if it is non-empty.
5. Return the resulting list.

In code terms:

```python
def _parse_visible_to(args):
    val = getattr(args, "visible_to", None)
    if val is not None:
        return [t.strip() for t in val.split(",") if t.strip()]
    return None
```

## Side Effects

None.

- Does not mutate `args`.
- Does not mutate `args.visible_to`.
- Does not perform I/O.
- Does not call the API.
- Allocates and returns a new list when parsing occurs.

## Error Handling

No explicit error handling is performed.

Possible failures:

- If `visible_to` is non-`None` and is not a string-like object with `.split()`, an `AttributeError` can occur.
- If `split()` returns elements without `.strip()`, an `AttributeError` can occur.
- No validation errors are raised for malformed tags.

For normal argparse usage, `visible_to` is either a string or missing/`None`, so the function is effectively safe.

## Usage Patterns

Typical usage is inside CLI command handlers:

```python
result = api.search(
    args.query,
    visible_to=_parse_visible_to(args),
    ...
)
```

or:

```python
node = api.show_node(
    args.node_id,
    visible_to=_parse_visible_to(args),
    ...
)
```

Callers rely on this helper to bridge:

- CLI input: comma-separated string
- API input: `None` or `list[str]`

Caller obligations:

- Do not assume `[]` and `None` are equivalent unless the API explicitly treats them that way.
- Handle API-level visibility failures such as `PermissionError` where appropriate.
- Ensure the relevant argparse subparser actually defines `--visible-to` if the caller wants the option to be accepted.

## Dependencies

- Built-in `getattr`
- Python string methods: `.split()` and `.strip()`
- Indirect dependency on argparse conventions: the parsed destination name is `visible_to`
- Indirect dependency on `reasons.api` functions accepting a `visible_to` parameter

The function itself does not import or reference any external modules.

## Topics to Explore

- [function] `reasons/cli.py:cmd_show` — Representative caller that parses `--visible-to`, passes it into `api.show_node`, and converts `PermissionError` into CLI output.
- [function] `reasons/api.py:show_node` — Shows how the API interprets `visible_to=None`, a non-empty list, and an empty list against node access tags.
- [file] `reasons/cli.py` — Where `--visible-to` is declared across many subparsers and how this helper is reused consistently.
- [file] `tests/test_cli.py` — Tests that exercise CLI parsing and access-tag visibility behavior.
- [general] `access-tag-subset-semantics` — The precise rule for “access_tags are a subset of visible_to” and whether an empty allowed-tag list restricts to untagged nodes.

## Beliefs

- `visible-to-none-means-no-filter` — `_parse_visible_to` returns `None` when the `visible_to` attribute is absent or `None`, which callers pass to APIs to disable access-tag filtering.
- `visible-to-empty-string-yields-empty-list` — An explicitly empty or whitespace-only `--visible-to` value returns `[]`, not `None`, so callers and APIs must handle the empty-list case separately.
- `visible-to-parsing-drops-empty-tokens` — Parsing splits on commas, strips surrounding whitespace, and drops empty tokens while preserving order and duplicates.
- `visible-to-parser-assumes-string-input` — The helper assumes any non-`None` `visible_to` value is a string; a non-string value without `.split()` raises `AttributeError`.
- `visible-to-parser-has-no-side-effects` — The helper does not mutate arguments or perform I/O; it returns a newly allocated list when parsing occurs.
