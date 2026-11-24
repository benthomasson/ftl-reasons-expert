---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: _linkify in reasons/build_wiki.py

# `_linkify`

## Purpose

`_linkify` post-processes a rendered Markdown page and replaces bare cross-page belief/node IDs with Markdown links of the form:

```markdown
[node-id](target-page.md#node-id)
```

It exists so pages can mention belief IDs as plain text while still producing navigable links to the page and anchor where that belief is documented.

It only links IDs whose target page is known and is different from the current page. Same-page references are intentionally left alone.

## Contract

### Preconditions

- `content` is a string containing Markdown/text.
- `current_page` is the filename of the page currently being built, e.g. `"topic-page.md"`.
- `node_to_page` is a mapping from node/belief ID to target Markdown filename.
- `all_ids` is an iterable of node/belief IDs, normally `node_details.keys()`.
- IDs are expected to be non-empty strings and are usually lowercase kebab-case identifiers.
- The caller has already assembled the full page text; `_linkify` does not understand Markdown structure.

### Postconditions

- Returns a string where eligible bare occurrences of IDs have been converted into Markdown links.
- Does not modify the input objects.
- Does not link:
  - IDs missing from `node_to_page`.
  - IDs whose target page equals `current_page`.
  - IDs that do not appear as a substring in `content`.
  - IDs for which `content` already contains an exact link label form `[nid](` anywhere.
- Processes IDs in descending string-length order.

### Invariants / assumptions

- The regex boundary only protects against adjacent lowercase ASCII letters, digits, and hyphens:
  - Lookbehind: `(?<![a-z0-9\-])`
  - Lookahead: `(?![a-z0-9\-])`
- Matching is case-sensitive.
- The function is not Markdown-aware. It scans the whole page text, including code blocks, existing URLs, generated link targets, etc.
- It assumes node IDs and target filenames are safe for use inside a Markdown link and as a `re.sub` replacement string.
- It does not validate that the target page actually exists on disk.

## Parameters

| Parameter | Meaning | Expected type | Edge cases |
|---|---|---|---|
| `content` | Full Markdown page text to transform. | `str` | `None` or non-string will raise later during `nid in content` or `re.sub`. |
| `current_page` | Filename of the page being built. Used to skip same-page links. | `str` | Compared directly to `node_to_page[nid]`. |
| `node_to_page` | Mapping from belief ID to page filename. | Mapping, usually `dict[str, str]` | Missing or falsy target causes skip. If target is not a string, link concatenation may raise `TypeError`. |
| `all_ids` | Candidate belief IDs to consider. | Iterable of strings | Sorted with `key=len`, so every element must support `len()`. Usually `node_details.keys()`. IDs not in `node_to_page` are skipped. |

## Return Value

Returns the transformed `content` string. If no replacements occur, it returns the original content object/value unchanged. The caller is expected to write or use the returned string.

## Algorithm

1. Sort `all_ids` by string length in descending order.
   - Longer IDs are considered first, reducing ambiguity when one ID might contain another.
2. For each `nid`:
   1. Look up `target = node_to_page.get(nid)`.
   2. Skip if there is no target or if `target == current_page`.
   3. Skip if `nid` is not present in `content` at all.
   4. Skip if `content` already contains the exact Markdown link prefix `"[nid]("`.
   5. Build the replacement link:
      ```python
      "[nid](target#nid)"
      ```
   6. Build a regex that matches `nid` only when it is not immediately adjacent to `[a-z0-9-]`:
      ```python
      r"(?<![a-z0-9\-])" + re.escape(nid) + r"(?![a-z0-9\-])"
      ```
   7. Replace every matching occurrence in `content` with the link using `re.sub`.
3. Return the final `content`.

Example: if `content` contains:

```markdown
The mechanism is described in sl-justification-mechanism.
```

and `sl-justification-mechanism` is not linked yet, and its target page is `logic.md`, it becomes:

```markdown
The mechanism is described in [sl-justification-mechanism](logic.md#sl-justification-mechanism).
```

## Side Effects

- No file I/O.
- No mutation of `node_to_page`, `all_ids`, or other caller-owned objects.
- Rebinds the local `content` variable to new strings returned by `re.sub`.
- Returns a new string when replacements occur; otherwise returns the original string value.

## Error Handling

`_linkify` does not catch or suppress exceptions. Normal inputs should not raise. Potential exceptions include:

- `TypeError` if `all_ids` is not iterable or contains values without `len()`.
- `TypeError` if `content` is not a string or if `node_to_page[nid]` is not a string when concatenating the link.
- `AttributeError` if `node_to_page` does not provide `.get`.
- `re.error` if the replacement string contains problematic backslash sequences. This is possible because `link` is passed directly as the `re.sub` replacement template and is not escaped. In practice, IDs and page filenames are expected to be simple kebab-case strings.
- `re.error` from invalid regex construction is unlikely because `nid` is passed through `re.escape`, but non-string IDs would still fail.

Missing targets, same-page targets, absent IDs, and already-linkified IDs are silently skipped. There is no return value indicating which IDs were linked or skipped.

## Usage Patterns

`_linkify` is called from `build_wiki` after assembling each page’s text and before writing the page file:

```python
page_text = _linkify(page_text, page_file, node_to_page, node_details.keys())
```

It is used for both LLM-generated page content and fallback `_format_node` content. The caller must supply:

- the final page text,
- the current page filename,
- a complete `node_to_page` mapping,
- all known IDs that should be considered for linking.

If `all_ids` omits an ID, that ID will never be linkified. If `node_to_page` omits an ID, it will be skipped. Same-page references are not created by this function.

## Important Behavioral Notes

- The check `if "[" + nid + "](" in content` is global per ID. If one exact link for `nid` exists anywhere, all bare occurrences of `nid` on the page are skipped.
- The regex boundary does not protect against uppercase letters, underscores, dots, slashes, parentheses, `#`, etc. It only treats lowercase ASCII letters, digits, and hyphens as “part of an identifier” for boundary purposes.
- Because the function scans the entire page text repeatedly, it can match IDs inside Markdown syntax, URLs, code spans, or previously inserted link targets if those occurrences satisfy the regex boundary.
- The replacement string is not passed through `re.escape`; only the pattern side is escaped. This assumes IDs and page filenames do not contain backslash sequences that Python `re.sub` would interpret as replacement escapes.
- The function does not recursively re-process inserted links within the same `re.sub` call, but later IDs can potentially match inside text inserted by earlier IDs if their boundary conditions allow it.

## Dependencies

Only the Python standard library module `re` is used directly by this function. It also uses built-ins: `sorted`, `len`, `str`, and `dict.get`.

## Topics to Explore

- [function] `reasons/build_wiki.py:build_wiki` — Shows how `_linkify` is called during page assembly and how `current_page` and `node_to_page` are constructed.
- [function] `reasons/build_wiki.py:_format_node` — Shows how existing Markdown links are generated before `_linkify` runs.
- [file] `tests/test_build_wiki.py` — Contains expected behavior and edge cases for wiki linkification.
- [general] `markdown-linkify-boundaries` — Explore how regex boundaries interact with Markdown links, URLs, code blocks, and mixed-case IDs.
- [general] `node-id-and-page-slug-generation` — Investigate whether node IDs can collide with page slugs or contain characters unsafe for regex replacement.

## Beliefs

- `linkify-requires-node-to-page-mapping` — `_linkify` only creates a link when `node_to_page.get(nid)` returns a truthy target page.
- `linkify-skips-current-page-targets` — `_linkify` skips any node whose target page equals `current_page`, so it does not create same-page links.
- `linkify-processes-ids-by-descending-length` — `_linkify` iterates over `all_ids` sorted by `len` in reverse order.
- `linkify-skips-id-if-exact-link-exists` — If `content` contains `"[nid]("`, `_linkify` skips all bare occurrences of that `nid`.
- `linkify-is-not-markdown-aware` — `_linkify` applies regex substitution to the full page text and may match IDs inside code, URLs, or existing Markdown syntax if boundary conditions allow.
