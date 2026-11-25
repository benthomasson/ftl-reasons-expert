---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Investigate path-component boundaries, query strings, fragments, and URL-encoded paths when using `str.endswith`.

## What the observations show

The only `str.endswith` usage in the provided data that relates to URLs is:

- `reasons/check_stale.py:302` — `if source_path and url.endswith(source_path):`

The other `endswith` hits are unrelated to URL path matching:

- `reasons/api.py:256` and `reasons/derive.py:151` — node ID suffix checks like `:active`
- `tests/test_metadata_extended.py:48` — ISO timestamp suffix check
- `tests/test_review.py:548` and `:597` — report filename suffix `.json`
- `tests/test_contradictions.py:76` — unrelated assertion placeholder

Also, the provided grep for URL parsing/component logic does not show `urlsplit`, `urlparse`, or `unquote` in `reasons/check_stale.py`. The only path-component-related hit is `reasons/check_stale.py:32`, which describes repo lookup by first path component, not URL matching.

So the relevant conclusion is: `reasons/check_stale.py:302` is using raw string suffix matching against a URL, without visible URL parsing or normalization.

## Why `url.endswith(source_path)` is unsafe

`str.endswith` is a byte/string operation. It knows nothing about URL structure, path hierarchy, query strings, fragments, or percent-encoding.

### 1. Path-component boundaries are not enforced

`url.endswith(source_path)` can match a suffix that is not aligned to whole path components.

Example:

```python
url = "https://example.com/notsrc/check_stale.py"
source_path = "src/check_stale.py"

url.endswith(source_path)  # True
```

But the actual URL path components are:

```text
["notsrc", "check_stale.py"]
```

not:

```text
["src", "check_stale.py"]
```

Similarly:

```python
url = "https://example.com/myapp.py"
source_path = "app.py"

url.endswith(source_path)  # True
```

The match is caused by string suffix overlap, not because `app.py` is a complete path component.

A robust comparison should compare path components, or at least require the character before the matched suffix to be `/` after parsing the URL path.

### 2. Query strings break suffix matching

A URL with a query string usually does not end with the filesystem path.

```python
url = "https://example.com/repo/src/check_stale.py?ref=main"
source_path = "src/check_stale.py"

url.endswith(source_path)  # False
```

The raw URL ends with `?ref=main`, so the suffix check fails even though the URL path does end with `src/check_stale.py`.

There is also a false-positive risk:

```python
url = "https://example.com/download?file=src/check_stale.py"
source_path = "src/check_stale.py"

url.endswith(source_path)  # True
```

Here the URL path is only `/download`; the match comes from the query value, not the path.

To handle this correctly, parse the URL first and compare against the `.path` component, not the full URL string.

### 3. Fragments break suffix matching the same way

Fragments are part of the URL string but not part of the URL path.

```python
url = "https://example.com/repo/src/check_stale.py#L10"
source_path = "src/check_stale.py"

url.endswith(source_path)  # False
```

Again, the match fails because the raw string ends with `#L10`.

Fragments can also create false positives:

```python
url = "https://example.com/page#src/check_stale.py"
source_path = "src/check_stale.py"

url.endswith(source_path)  # True
```

But the URL path is `/page`.

### 4. URL-encoded paths will not match raw filesystem paths

URLs often percent-encode spaces, brackets, Unicode characters, etc.

```python
url = "https://example.com/repo/My%20File.md"
source_path = "repo/My File.md"

url.endswith(source_path)  # False
```

Even though the decoded URL path matches, the raw string does not because `%20` is not `" "`.

The reverse can also happen: a source path containing literal `%20` could incorrectly match a decoded or encoded form depending on which side is normalized.

The important point is that you need to define and normalize the comparison domain. Usually that means:

1. Parse the URL.
2. Use only the URL path.
3. Percent-decode path components.
4. Normalize `source_path` separators.
5. Compare path components, not raw string suffixes.

Be careful with encoded slashes. `%2F` is not necessarily a path separator in URL syntax. Decoding the whole path before splitting can change semantics. A safer approach is to split the raw URL path on `/`, then decode each component individually.

## Recommended shape for the check

The current line:

```python
if source_path and url.endswith(source_path):
```

should not operate on the raw URL. A safer function would look roughly like this:

```python
from urllib.parse import urlsplit, unquote
from pathlib import PurePosixPath


def url_path_ends_with_source_path(url: str, source_path: str) -> bool:
    parsed = urlsplit(url)
    url_parts = [unquote(part) for part in parsed.path.split("/") if part not in ("", ".")]
    source_parts = [
        unquote(part)
        for part in source_path.replace("\\", "/").split("/")
        if part not in ("", ".")
    ]

    if not source_parts or len(url_parts) < len(source_parts):
        return False

    return url_parts[-len(source_parts):] == source_parts
```

This handles:

- path-component boundaries
- query strings, because `urlsplit(url).path` excludes them
- fragments, because `urlsplit(url).path` excludes them
- percent-encoded path components, via `unquote`
- mixed slash styles in `source_path` by normalizing `\` to `/`

You would still need to decide policy for:

- absolute vs relative `source_path`
- trailing directory slashes
- `..` normalization
- encoded slash semantics
- case sensitivity
- whether `source_path` is a file path, directory path, or repo-relative path

## What is missing from the observations

The provided data does not include the surrounding function in `reasons/check_stale.py` around line 302. So we cannot tell whether `url` or `source_path` is normalized earlier, what `source_path` represents exactly, or whether this check is intended to match a file, directory, or repo root.

The test grep result also does not show direct coverage for this URL suffix behavior. The visible test `endswith` hits cover timestamps and `.json` report filenames, not query strings, fragments, or encoded URL paths.

## Topics to Explore

- [file] `reasons/check_stale.py` — Inspect the full function around line 302 to see how `url` and `source_path` are constructed and whether either is normalized before the suffix check.
- [general] `url-suffix-matching-tests` — Add tests for query strings, fragments, encoded paths, trailing slashes, and path-boundary false positives.
- [general] `urlsplit-vs-urlparse-and-unquote` — Explore `urllib.parse.urlsplit`, `unquote`, and component-wise comparison instead of raw `str.endswith`.
- [file] `tests/test_review.py` — Confirm that the `.json` `endswith` usages are unrelated filename filtering and not a pattern to reuse for URL matching.
- [general] `path-normalization-policy` — Decide how to treat relative source paths, encoded slashes, trailing directory slashes, and path separators across operating systems.

## Beliefs

- `check-stale-uses-raw-url-endswith` — `reasons/check_stale.py:302` checks URL/source-path matching with raw `str.endswith`, not parsed URL components.
- `raw-url-endswith-fails-with-query-or-fragment` — Any query string or fragment after the path causes `url.endswith(source_path)` to return false unless `source_path` includes that suffix.
- `raw-url-endswith-can-false-positive-on-query-value` — A query value ending with `source_path` can make `url.endswith(source_path)` return true even when the URL path does not match.
- `check-stale-grep-shows-no-url-unquoting` — The provided grep results show no `urlsplit`, `urlparse`, or `unquote` usage in `reasons/check_stale.py`.
- `existing-endswith-tests-do-not-cover-url-suffixes` — The visible test `endswith` hits cover ISO timestamps and `.json` report filenames, not URL query/fragment/encoded-path behavior.
