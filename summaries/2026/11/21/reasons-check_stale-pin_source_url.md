---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: pin_source_url in reasons/check_stale.py

## `pin_source_url`

### Purpose

`pin_source_url` rewrites a GitHub **blob** URL so that it points to a specific commit SHA instead of a branch/ref:

```text
https://github.com/<owner>/<repo>/blob/<branch>/<path>
    ->
https://github.com/<owner>/<repo>/blob/<sha>/<path>
```

It exists for the git-aware pinning flow: when a node is pinned to the current file commit, its `source_url` metadata should also be commit-stable, so readers see the exact file revision the node came from.

The function is a pure string transformation. It does not call GitHub, git, or the filesystem.

---

### Contract

#### Preconditions

- `url` is expected to be a string.
- `url` should ideally look like:

  ```text
  https://github.com/<owner>/<repo>/blob/<ref>/<path>
  ```

- `sha` is expected to be a commit SHA string, but no validation is performed.
- `source_path`, if provided, is intended to be the repository-relative file path that should replace the ref portion, e.g. `docs/design.md`.
- The implementation only uses `source_path` if the full URL literally ends with it:

  ```python
  url.endswith(source_path)
  ```

  So `source_path` must be an exact URL suffix, not merely a path that appears somewhere in the URL.

#### Postconditions

- If `url` does not match the expected GitHub blob prefix, the original `url` is returned unchanged.
- If `url` matches but there is no slash after the inferred ref segment, the original `url` is returned unchanged.
- If `source_path` is non-empty and is an exact URL suffix, the result is:

  ```text
  https://github.com/<owner>/<repo>/blob/<sha>/<source_path>
  ```

  Everything between `/blob/` and `source_path` is discarded as the ref.

- Otherwise, the fallback path replaces only the first segment after `/blob/` with `sha` and preserves the rest:

  ```text
  https://github.com/<owner>/<repo>/blob/<sha>/<remaining>
  ```

#### Invariants

- The scheme, host, owner, repo, and `/blob/` prefix are preserved.
- The function never mutates its arguments.
- It performs no I/O and has no side effects.
- A malformed or unsupported URL is generally treated as a no-op.
- The function does not verify that `sha` is valid, that the URL exists, or that the repo/path are consistent.

---

### Parameters

#### `url: str`

The GitHub blob URL to rewrite.

Expected examples:

```text
https://github.com/acme/widgets/blob/main/docs/spec.md
https://github.com/acme/widgets/blob/feature/new-ui/docs/spec.md
```

Edge cases:

- Non-GitHub URLs return unchanged.
- `http://` URLs return unchanged, because the regex requires `https://`.
- GitHub Enterprise URLs return unchanged.
- `tree` URLs return unchanged; only `/blob/` is matched.
- URLs without any character after `/blob/` return unchanged.
- URLs where the ref has no following slash return unchanged via the fallback branch.
- Query strings and fragments are not parsed specially. In the fallback path they are preserved as part of the remaining string.

#### `sha: str`

The commit SHA to insert after `/blob/`.

The function treats this opaquely. It does not check:

- length
- hexadecimal format
- whether it exists
- whether it belongs to the same repository as the URL

If `sha` is empty, the result can contain `.../blob//...`.

#### `source_path: str = ""`

Optional known source path used to disambiguate branch names containing slashes.

Example:

```python
pin_source_url(
    "https://github.com/acme/widgets/blob/feature/new-ui/docs/spec.md",
    "abc123",
    "docs/spec.md",
)
# -> https://github.com/acme/widgets/blob/abc123/docs/spec.md
```

The condition is strict:

```python
if source_path and url.endswith(source_path):
```

So this only works when `source_path` is an exact suffix of `url`. If `source_path` is `src/spec.md` but the URL ends with `docs/src/spec.md`, it does not match unless the URL suffix is exactly `src/spec.md`; note that this can also match inside a larger path component because `endswith` is only a string check.

If `source_path` is empty or does not match, the fallback heuristic is used.

---

### Return Value

Returns a string.

- If no rewrite is possible, returns the original `url`.
- If `source_path` matches, returns the rebuilt URL with `sha` and `source_path`.
- Otherwise, if the fallback can split into ref and remainder, returns the URL with the first segment after `/blob/` replaced by `sha`.
- Otherwise, returns the original `url`.

The caller cannot distinguish “successfully rewritten” from “returned unchanged” just by the return type. It must compare strings or use its own knowledge of whether a rewrite was expected.

---

### Algorithm

1. Match the URL against:

   ```python
   r'(https://github\.com/[^/]+/[^/]+/blob/).+'
   ```

   This captures the prefix:

   ```text
   https://github.com/<owner>/<repo>/blob/
   ```

   The `.+` requires at least one character after `/blob/`.

   Because `re.match` is used, the URL must start with this prefix. Because the pattern is not anchored at the end, the rest of the URL can contain anything.

2. If there is no match, return `url` unchanged.

3. Save the captured prefix as `prefix`.

4. If `source_path` is truthy and `url.endswith(source_path)`, rebuild the URL:

   ```python
   f"{prefix}{sha}/{source_path}"
   ```

   This is the slash-safe branch-name path. It discards whatever ref text appeared between `/blob/` and `source_path`.

   For example:

   ```text
   url         = https://github.com/o/r/blob/feature/foo/docs/a.md
   sha         = 123abc
   source_path = docs/a.md
   result      = https://github.com/o/r/blob/123abc/docs/a.md
   ```

5. Otherwise, extract the part after the captured prefix:

   ```python
   rest = url[len(prefix):]
   ```

   For example, for:

   ```text
   https://github.com/o/r/blob/main/docs/a.md
   ```

   `rest` is:

   ```text
   main/docs/a.md
   ```

6. Split `rest` once on `/`:

   ```python
   parts = rest.split("/", 1)
   ```

7. If there are two parts, replace the first part, assumed to be the ref, with `sha` and keep the second part:

   ```python
   return f"{prefix}{sha}/{parts[1]}"
   ```

   Example:

   ```text
   rest  = main/docs/a.md
   parts = ["main", "docs/a.md"]
   result = https://github.com/o/r/blob/<sha>/docs/a.md
   ```

8. If there is no slash after the ref, return the original `url`.

This fallback assumes the first segment after `/blob/` is the complete ref. That is true for branches like `main`, but false for branches like `feature/foo`. That is exactly why the `source_path` shortcut exists.

---

### Side Effects

None.

- No file reads or writes.
- No git commands.
- No network calls.
- No mutation of `url`, `sha`, or `source_path`.
- No logging.

The caller is responsible for assigning the result, e.g. inside `pin_sources`:

```python
node.source_url = pin_source_url(node.source_url, sha, node.source)
```

---

### Error Handling

The function uses a fail-open string-oriented style:

- Unsupported URL shapes return the input unchanged.
- URLs with no path after the ref return unchanged.
- There is no explicit exception handling for malformed inputs.

Possible exceptions come from Python type errors:

- If `url` is `None` or another non-string, `re.match` can raise `TypeError`.
- If `source_path` is truthy but not a valid argument to `str.endswith`, `url.endswith(source_path)` can raise `TypeError`.
- If `source_path` is `None`, the truthiness check skips it and no error occurs.

The function does not validate:

- SHA format
- GitHub URL validity beyond the regex
- repository ownership
- path existence
- whether `source_path` belongs to `url` at a path-component boundary

---

### Usage Patterns

The in-file caller is `pin_sources`.

When `pin_urls=True`, `pin_sources` calls:

```python
if pin_urls and node.source_url:
    node.source_url = pin_source_url(node.source_url, sha, node.source)
```

At that point:

- `sha` comes from `get_file_commit_sha(path)`, i.e. the last commit that touched the resolved source file.
- `source_path` is `node.source`.
- `source_url` is updated only if it exists.

Caller obligations:

- Pass a SHA associated with the same repository as the URL.
- For slash-containing branch refs, pass a `source_path` that is an exact URL suffix matching the file path.
- Do not assume the URL was changed; malformed or unsupported URLs are returned unchanged.
- The function only updates the URL string. It does not update `pinned_sha`, `source_hash`, or `verified_at`; those are handled by the caller.

---

### Dependencies

Only the Python standard library `re` module is used.

No internal module functions are called from `pin_source_url`. It is independent of the git helpers and `Network` object in the same file.

---

### Important Assumptions Not Enforced by the Type System

- The URL must be a GitHub.com blob URL, not a GitHub Enterprise, raw, tree, or API URL.
- The first two path components after the host are assumed to be owner and repo.
- The fallback branch assumes the ref contains no slash.
- The `source_path` branch assumes `source_path` is an exact URL suffix and represents the full file path portion.
- `url.endswith(source_path)` is only a string suffix check; it does not require a `/` component boundary.
- `sha` is assumed to be a valid commit SHA but can be any string.
- Query strings and fragments are not parsed; behavior depends on whether they prevent `endswith(source_path)`.
- The function never confirms that the URL’s owner/repo match the repository used to compute `sha`.

---

## Topics to Explore

- [file] `tests/test_pin_sources.py` — Test cases around URL pinning, branches with slashes, and expected no-op behavior.
- [function] `reasons/check_stale.py:pin_sources` — Shows how `pin_source_url` is called and how SHA pinning interacts with node metadata.
- [function] `reasons/check_stale.py:resolve_source_path` — Explains how `node.source` is mapped to filesystem paths and why its format matters for URL suffix matching.
- [general] `github-slash-ref-handling` — Explore GitHub URL conventions for branch names with slashes and why simple first-segment replacement is insufficient.
- [general] `url-suffix-matching-edge-cases` — Investigate path-component boundaries, query strings, fragments, and URL-encoded paths when using `str.endswith`.

## Beliefs

- `pin-source-url-is-pure` — It performs only string operations and has no I/O, mutation, or global state changes.
- `pin-source-url-returns-input-for-unsupported-urls` — URLs not matching the exact `https://github.com/<owner>/<repo>/blob/` prefix or lacking a slash after the ref are returned unchanged.
- `pin-source-url-prefers-source-path-suffix` — When `source_path` is non-empty and is an exact suffix of `url`, it rebuilds the URL as `.../blob/<sha>/<source_path>` and discards the original ref segment.
- `pin-source-url-fallback-assumes-slashless-ref` — Without a matching `source_path`, it replaces only the first segment after `/blob/` with `sha`, which mispins slash-containing branch refs.
- `pin-source-url-does-not-validate-sha-or-url` — It does not verify SHA format, URL existence, repository consistency, or that `source_path` starts at a path-component boundary.
