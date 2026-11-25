---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Explore GitHub URL conventions for branch names with slashes and why simple first-segment replacement is insufficient.

## GitHub URL shape and ambiguity

GitHub blob URLs look like:

```text
https://github.com/{owner}/{repo}/blob/{ref}/{path}
```

Here `{ref}` can be a branch, tag, or commit SHA. Branch names may legally contain `/`, for example `feature/fix`. GitHub does not percent-encode those slashes in the web URL, so a URL like:

```text
https://github.com/owner/repo/blob/feature/fix/src/main.py
```

is ambiguous to a pure string parser. It could mean:

- branch `feature/fix`, file path `src/main.py`
- branch `feature`, file path `fix/src/main.py`

GitHub resolves this server-side by looking up refs, but a client-side URL rewriter cannot know the boundary from the URL alone.

## Evidence in this codebase

`tests/test_pin_sources.py` defines the expected behavior:

- Lines 95–98: a normal branch `main` is replaced with a SHA.
- Lines 106–109: a branch without slashes, `feature-branch`, is replaced.
- Lines 111–115: a branch with a slash, `feature/fix`, must become:

```text
https://github.com/owner/repo/blob/{sha}/src/main.py
```

That expectation means the branch is `feature/fix`, not `feature`, and the source path is `src/main.py`. A naive first-segment replacement would produce:

```text
https://github.com/owner/repo/blob/{sha}/fix/src/main.py
```

which points at the wrong path.

`reasons/check_stale.py` explicitly acknowledges this problem:

- Lines 293–296: the docstring says the function rewrites a GitHub blob URL from a branch ref to a commit SHA and “handles branch names containing slashes.”
- Line 298: the regex captures only the stable prefix:

```python
m = re.match(r'(https://github\.com/[^/]+/[^/]+/blob/).+', url)
```

Everything after `/blob/` is the ambiguous part.

- Line 305 contains `parts = rest.split("/", 1)`. That operation, by itself, is exactly the insufficient first-segment split: it would treat `feature` as the branch and `fix/src/main.py` as the remaining path.

## Why first-segment replacement is insufficient

A simple implementation might do:

```python
prefix, rest = url.split("/blob/", 1)
branch, path = rest.split("/", 1)
return prefix + "/blob/" + sha + "/" + path
```

For `feature/fix/src/main.py`, this gives:

```text
branch = "feature"
path = "fix/src/main.py"
```

and rewrites to:

```text
.../blob/{sha}/fix/src/main.py
```

But the test at `tests/test_pin_sources.py:111-115` requires:

```text
.../blob/{sha}/src/main.py
```

So the rewrite must disambiguate the ref boundary using additional context. Likely options are:

- an explicit source path argument, as suggested by the test name `test_rewrites_slashed_branch_with_source_path`
- a known list of refs from GitHub’s API
- a local git repository lookup
- matching candidate branch names against the URL path

The observations do not include the full body of the function around `reasons/check_stale.py:293-305`, so the exact disambiguation mechanism is missing. We can see that line 305 uses `rest.split("/", 1)`, but we cannot tell whether additional logic before or after that line supplies the source path or ref information needed to handle `feature/fix` correctly.

## Topics to Explore

- [file] `reasons/check_stale.py` — Read the full URL-pinning function around lines 293–305 to see how it actually disambiguates slashed branch names.
- [function] `tests/test_pin_sources.py:test_rewrites_slashed_branch_with_source_path` — Shows the exact input/output contract for `feature/fix/src/main.py` and whether a source path is supplied.
- [general] `github-ref-resolution` — Understand how GitHub resolves ambiguous `/blob/` URLs when branch names contain slashes.
- [general] `source-url-pinning-strategy` — Compare using an explicit source path, GitHub API ref lookup, or local git refs to rewrite branch URLs to SHAs.
- [file] `tests/test_pin_sources.py` — All branch-rewrite cases define the expected behavior for `main`, `feature-branch`, and `feature/fix`.

## Beliefs

- `github-blob-urls-are-ambiguous-with-slashed-refs` — A GitHub `/blob/` URL cannot be split reliably into ref and file path using only `/` when branch names contain slashes.
- `test-pin-sources-defines-slashed-branch-behavior` — `tests/test_pin_sources.py:111-115` expects `feature/fix/src/main.py` to rewrite to `/blob/{sha}/src/main.py`.
- `check-stale-docstring-claims-slashed-branch-support` — `reasons/check_stale.py:293-296` states that the rewrite handles branch names containing slashes.
- `check-stale-captures-prefix-through-blob` — The regex at `reasons/check_stale.py:298` captures only `https://github.com/{owner}/{repo}/blob/`, leaving the ambiguous remainder.
- `first-segment-split-alone-is-insufficient` — The `rest.split("/", 1)` at `reasons/check_stale.py:305` would treat only the first segment as the ref unless additional context is used.
