---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: _page_name in reasons/build_wiki.py

# `_page_name`

## Purpose

`_page_name` converts an arbitrary topic or cluster label into a slug suitable for use as a markdown filename base. It is used by `build_wiki` to map group labels such as `"Hello, World!"` or `"C++"` to stable, filesystem-friendly names like `"hello-world"` and `"c"`.

The function only produces the slug. The caller is responsible for appending `.md`, resolving collisions, and avoiding reserved filenames such as `index.md`.

## Contract

### Preconditions

- `label` is expected to be a `str`.
- No other type validation is performed.
- The label may contain uppercase letters, punctuation, whitespace, Unicode, path separators, etc.

### Postconditions

- The return value is always a non-empty string.
- The return value consists only of lowercase ASCII letters, digits, and single internal hyphens.
- The return value never starts or ends with `-`.
- The return value never contains consecutive hyphens.
- The return value may be `"other"` when sanitization produces an empty string.
- The return value is not guaranteed to be unique.
- The return value is not guaranteed to avoid reserved names such as `"index"`.
- The return value does not include a file extension.

### Invariant

For any normal string input, the returned slug matches:

```text
[a-z0-9]+(-[a-z0-9]+)*
```

## Parameters

- `label: str` — The topic or cluster label to sanitize.

Edge cases:

- Empty string `""` → `"other"`
- Only punctuation / whitespace / Unicode → `"other"`
- Uppercase ASCII → lowercased
- Multiple non-alphanumeric characters → collapsed to one hyphen
- Leading/trailing non-alphanumeric characters → stripped after replacement
- Path separators such as `/` and `\` → replaced with hyphens, so the result is not a path
- Long labels → not truncated, so generated filenames may exceed filesystem limits
- `None`, `bytes`, integers, etc. → not handled explicitly and will usually raise `AttributeError` or `TypeError`

Examples:

```python
_page_name("Hello, World!")     # "hello-world"
_page_name("  Foo__Bar  ")      # "foo-bar"
_page_name("C++")               # "c"
_page_name("!!!")               # "other"
_page_name("é")                 # "other"
_page_name("Index")             # "index"
_page_name("A B")               # "a-b"
_page_name("a-b")               # "a-b"
```

Note that `"A B"` and `"a-b"` both map to `"a-b"`, so uniqueness is not guaranteed.

## Return Value

Returns a sanitized slug string.

The caller must handle:

- appending `.md`
- collision resolution between duplicate slugs
- reserved names such as `index`
- possible overlong filenames
- platform-specific reserved names such as Windows device names like `CON`, `NUL`, etc.

In `build_wiki`, these responsibilities are handled via `_RESERVED_SLUGS` and the `used_slugs` dictionary.

## Algorithm

1. Lowercase the input:
   ```python
   label.lower()
   ```

2. Replace every maximal run of characters outside lowercase ASCII letters and digits with a single hyphen:
   ```python
   re.sub(r'[^a-z0-9]+', '-', label.lower())
   ```

3. Strip leading and trailing hyphens:
   ```python
   .strip('-')
   ```

4. If the resulting string is empty, return `"other"`:
   ```python
   return safe or "other"
   ```

The `+` in the regex is important: any contiguous block of invalid characters becomes one `-`, not multiple hyphens.

## Side Effects

None.

`_page_name` is a pure function:

- no file I/O
- no mutation of arguments
- no global state changes
- no external calls

It does not create directories or write markdown files despite dealing with filenames.

## Error Handling

There is no explicit error handling.

Possible exceptions:

- `AttributeError` if `label` is not string-like and lacks `.lower()`, e.g. `None` or `123`
- `TypeError` if `label.lower()` returns something incompatible with the string regex, e.g. bytes input

Invalid but string-valued content does not raise. For example, `"!!!"` simply falls back to `"other"`.

The caller `build_wiki` does not catch exceptions from `_page_name`, so a non-string group label would fail the wiki build.

## Usage Patterns

Typical usage in `build_wiki`:

```python
for label in groups:
    slug = _page_name(label)
    if slug in _RESERVED_SLUGS:
        slug = f"{slug}-topic"
    while slug in used_slugs:
        slug = f"{slug}-2"
    used_slugs[slug] = label
    label_to_file[label] = slug + ".md"
```

Caller obligations:

- append the `.md` extension
- ensure the slug is unique
- avoid reserved filenames such as `"index"`
- keep the original label separately for display headings and index links

The function is private to the module by naming convention (`_page_name`), so it should generally not be treated as a public API.

## Dependencies

- Standard library `re`, imported at module top level.
- Used by `build_wiki` in `reasons/build_wiki.py`.
- Related tests are in `tests/test_build_wiki.py`.

No third-party dependencies.

## Topics to Explore

- [file] `tests/test_build_wiki.py` — Tests likely cover slug sanitization, fallback behavior, and filename collision handling.
- [function] `reasons/build_wiki.py:build_wiki` — Shows how `_page_name` feeds into `label_to_file`, reserved slug handling, and page generation.
- [general] `slug-collision-handling` — Explore how `used_slugs` resolves duplicate slugs and why `_page_name` itself does not guarantee uniqueness.
- [general] `unicode-label-policy` — Understand how non-ASCII labels collapse to `other` and whether that is acceptable for topic grouping.
- [function] `reasons/build_wiki.py:_linkify` — Cross-reference links depend on the generated page filenames produced from these slugs.

## Beliefs

- `_page_name-output-invariant` — `_page_name` returns a non-empty slug containing only lowercase ASCII letters, digits, and single internal hyphens.
- `_page_name-collapses-non-alnum-runs` — Consecutive characters outside `[a-z0-9]` are replaced by a single hyphen before leading and trailing hyphens are stripped.
- `_page_name-does-not-guarantee-uniqueness` — Distinct labels can map to the same slug, so `build_wiki` must resolve duplicate filenames using `used_slugs`.
- `_page_name-assumes-string-input` — Passing `None`, bytes, or other non-string values is not handled and will typically raise an attribute or type error.
- `_page_name-ignores-unicode` — Non-ASCII characters are treated as separators, and labels composed only of non-ASCII characters fall back to `"other"`.
