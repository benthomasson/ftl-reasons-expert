---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: resolve_repo_id in reasons/hf.py

## Purpose

`resolve_repo_id` normalizes a HuggingFace repository identifier into a consistent string before it is used in API URLs or repo-creation calls. It exists so callers can accept:

- a bare repo name like `ddia-expert`
- an explicit `org/repo` identifier
- a full HuggingFace URL

and have it converted into a usable repo id.

For bare names it prepends a default organization: `EEM-Hub`, unless overridden by `REASONS_HF_ORG`. For explicit ids and HuggingFace URLs it avoids adding the default org.

## Contract

### Preconditions

- `repo_id` is expected to be a `str`.
- The type system does not enforce that `repo_id` is non-`None`; passing `None` will fail at `.strip()`.
- For HTTP(S) inputs, the string is expected to contain the literal substring `huggingface.co/`.
- For non-URL inputs containing `/`, callers are expected to provide a valid repo identifier shape, typically `org/repo`.

### Postconditions

- Leading/trailing whitespace is removed via `.strip()`.
- Trailing `/` characters are removed via `.rstrip("/")`.
- If the cleaned input starts with `http://` or `https://`:
  - it must contain `huggingface.co/`;
  - the returned value is everything after the first occurrence of `huggingface.co/`.
- If the cleaned input contains `/` and is not a URL:
  - it is returned unchanged after trimming.
- If the cleaned input contains no `/`:
  - it is prefixed with `REASONS_HF_ORG` if set, otherwise `EEM-Hub`.

### Invariants / assumptions not enforced by types

- The function does not validate that the result is a valid HuggingFace repo id.
- The function does not guarantee exactly two slash-separated components.
- URL parsing is substring-based, not host-based:
  - any string starting with `http://` or `https://` and containing `huggingface.co/` anywhere will be accepted;
  - the host is not parsed or verified as a real HuggingFace domain.
- The domain match is case-sensitive: `HUGGINGFACE.co` will not match.
- Environment variable values are not sanitized. An empty `REASONS_HF_ORG` produces a leading slash, and a value with a trailing slash produces a double slash.
- Whitespace exposed by removing a trailing slash is not re-stripped. For example, `"repo /"` becomes `"repo "` before default-org handling.

## Parameters

### `repo_id: str`

The repository identifier to normalize. Expected forms:

- `"ddia-expert"` → bare name; default org is prepended.
- `"myuser/my-eem"` → explicit org/repo; returned as-is after trimming.
- `"https://huggingface.co/org/repo"` → converted to `"org/repo"`.
- `"https://huggingface.co/org/repo/"` → trailing slash removed, then converted to `"org/repo"`.

Edge cases:

- `"  ddia-expert  "` → `"EEM-Hub/ddia-expert"` by default.
- `"org/repo/"` → `"org/repo"`.
- `"org/"` → `"org"` after trailing slash removal, so it is treated as a bare name and becomes `"EEM-Hub/org"`.
- `""` → `"EEM-Hub/"` by default.
- `"https://huggingface.co/org"` → returns `"org"`, which may violate downstream assumptions that the result contains `org/repo`.
- `"https://github.com/org/repo"` → raises `ValueError`.
- `"https://huggingface.co/"` → the trailing slash is stripped, leaving `"https://huggingface.co"`, which no longer contains `"huggingface.co/"`, so it raises `ValueError`.

## Return Value

Returns a `str`.

Return cases:

- URL containing `huggingface.co/`: returns the substring after the first `huggingface.co/`.
- Non-URL containing `/`: returns the cleaned input.
- Non-URL without `/`: returns `f"{default_org}/{repo_id}"`.

The caller must handle:

- `ValueError` for HTTP(S) strings that do not contain `huggingface.co/`.
- Potentially malformed repo ids because this function does not validate shape.
- Downstream assumptions such as “the result contains exactly one slash and two non-empty components” are not guaranteed by this function.

## Algorithm

1. Clean the input:
   ```python
   repo_id = repo_id.strip().rstrip("/")
   ```
   This removes surrounding whitespace, then removes all trailing `/` characters.

2. Check whether the cleaned input is an HTTP(S) URL:
   ```python
   if repo_id.startswith(("http://", "https://")):
   ```
   If so, split on the literal substring `"huggingface.co/"` at most once:
   ```python
   parts = repo_id.split("huggingface.co/", 1)
   ```
   - If two parts are produced, return the second part.
   - Otherwise, raise `ValueError`.

3. If the cleaned input contains any slash:
   ```python
   if "/" in repo_id:
       return repo_id
   ```
   This treats the input as already having an explicit organization or path.

4. Otherwise, treat it as a bare repo name:
   ```python
   default_org = os.environ.get("REASONS_HF_ORG", DEFAULT_HF_ORG)
   return f"{default_org}/{repo_id}"
   ```

## Side Effects

- Reads process environment state via `os.environ.get("REASONS_HF_ORG", ...)`.
- The environment lookup happens on every call, not at import time.
- No file I/O, network I/O, mutation of global state, or caching.
- Strings are immutable; the input is not mutated.

## Error Handling

- Raises `ValueError` if the input starts with `http://` or `https://` but does not contain the literal substring `"huggingface.co/"`.
- Does not catch or wrap any exceptions.
- Does not raise for malformed non-URL ids such as `""`, `"org//repo"`, `"/repo"`, or `"org/repo/extra"`.
- Passing a non-string such as `None` will raise an `AttributeError` from `.strip()`, despite the type annotation.

## Usage Patterns

This function is used by other functions in `reasons/hf.py`:

- `_parse_repo_id` delegates directly to `resolve_repo_id`.
- `download_network` uses the result to construct a download URL.
- `create_repo` uses the result and then immediately splits it:
  ```python
  org, name = parsed_id.split("/", 1)
  ```
  This caller assumes the resolved id contains a slash.
- `upload_file` uses the result inside the HuggingFace upload API URL.

Typical caller expectations:

- Bare names are convenient and get the default org automatically.
- Full HuggingFace URLs can be pasted directly.
- Explicit `org/repo` values are preserved.
- Callers that require a strict two-part `org/repo` id should validate the result themselves.

## Dependencies

- `os` for environment variable access.
- `DEFAULT_HF_ORG` from the same module, currently `"EEM-Hub"`.
- No HuggingFace SDK dependency; this module uses raw `urllib` elsewhere.

## Topics to Explore

- [file] `tests/test_hf_default_org.py` — Confirms default-org and `REASONS_HF_ORG` override behavior.
- [function] `reasons/hf.py:create_repo` — Shows a caller that assumes the resolved id splits cleanly into `org/name`.
- [file] `reasons/hf.py` — Full download/publish flow that consumes the resolved repo id.
- [general] `huggingface-repo-id-validation` — Explore HuggingFace’s actual repo-id constraints, which this function does not enforce.
- [general] `REASONS_HF_ORG-configuration` — Understand edge cases like empty values or trailing slashes in the default org override.

## Beliefs

- `resolve_repo_id-prepends-default-org` — Bare repo names are prefixed with `REASONS_HF_ORG` or `EEM-Hub`.
- `resolve_repo_id-requires-hf-url-substring` — HTTP(S) inputs must contain the literal substring `huggingface.co/` or `ValueError` is raised.
- `resolve_repo_id-preserves-explicit-slash-paths` — Non-URL inputs containing `/` are returned after trimming rather than validated or rewritten.
- `resolve_repo_id-reads-env-at-call-time` — The default org is resolved from the environment on each call, not cached at import time.
- `resolve_repo_id-does-not-validate-repo-shape` — The returned value is not guaranteed to be a valid or exactly two-component HuggingFace repo id.
