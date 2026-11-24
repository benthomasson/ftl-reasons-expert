---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: upload_file in reasons/hf.py

# `upload_file` (`reasons/hf.py`)

## Purpose

`upload_file` publishes a single raw byte payload to an existing HuggingFace **model** repository using the Hub’s low-level HTTP upload API. It is a small `urllib`-based helper, so it avoids depending on `huggingface_hub` and reuses this module’s `resolve_repo_id` normalization for bare repo names, explicit `org/repo` IDs, and HuggingFace URLs.

It is the write-side counterpart to `download_network`, but it is more general: it uploads arbitrary bytes to an arbitrary repository path.

## Contract

### Preconditions

- `repo_id` must resolve to a valid HuggingFace model repo identifier.
- The repo must already exist; `upload_file` does **not** call `create_repo`.
- The target branch is always `main`; the repo must be able to accept an upload there.
- `path_in_repo` should be a safe relative URL path within the repo, e.g. `network.json` or `metadata/info.json`.
- `content` must be bytes-like content accepted by `urllib.request.Request` as request body data.
- `token` must be a non-empty HuggingFace token with write permission for the repo.
- The target repo must be a **model** repo, because the URL hardcodes `/api/models/...`.

### Postconditions

On success:

- The remote file at `main/{path_in_repo}` is created or overwritten with `content`.
- The function returns `None`.
- It does not return commit metadata or the remote URL.

On failure:

- `HTTPError` responses are converted to `RuntimeError`.
- Other errors, such as non-HTTP transport errors, may propagate unchanged.
- Because this is a network PUT, an exception does not guarantee the server did not partially process the upload; a timeout after the server accepts the request may still leave the remote file changed.

### Invariants

- URL is always built as  
  `https://huggingface.co/api/models/{parsed_id}/upload/main/{path_in_repo}`
- HTTP method is always `PUT`.
- `Content-Type` is always `application/octet-stream`.
- Authorization header is always `Authorization: Bearer {token}`.
- The function never resolves a token from `HF_TOKEN` or the cached HuggingFace token file; unlike `download_network`, it uses the explicit `token` parameter only.

### Assumptions not enforced by the type system

- `parsed_id` is expected to be exactly `org/repo`, but `resolve_repo_id` can return values with extra path segments if given a URL like `https://huggingface.co/org/repo/tree/main`.
- `path_in_repo` is **not URL-encoded**. Spaces, `#`, `?`, `%`, etc. are interpolated directly into the URL.
- `path_in_repo` is not checked for leading `/`, `../`, empty value, or other path oddities.
- `token` is not checked for emptiness, whitespace, newlines, or validity.
- `content` is fully held in memory; there is no streaming or LFS handling.
- The 60-second timeout may be too short for large files.
- A 2xx response is treated as success without inspecting the response body for application-level errors.

## Parameters

### `repo_id: str`

HuggingFace repo identifier. It is passed through `resolve_repo_id`, which:

- strips whitespace and trailing `/`
- converts a HuggingFace URL to `org/repo`
- leaves explicit `org/repo` values unchanged
- prepends the default org for bare names

Examples:

- `"ddia-expert"` → `"EEM-Hub/ddia-expert"` by default
- `"myuser/my-eem"` → unchanged
- `"https://huggingface.co/org/repo"` → `"org/repo"`

Edge cases:

- A non-HuggingFace `http(s)` URL raises `ValueError` inside `resolve_repo_id`.
- An explicit repo ID containing `/` is returned as-is, even if it has extra path components.
- An empty `repo_id` is not rejected by this function and may produce an invalid repo ID such as `"EEM-Hub/"`.

### `path_in_repo: str`

Destination path inside the repository. Examples: `"network.json"`, `"README.md"`, `"config/info.json"`.

Important details:

- It is inserted directly into the URL:  
  `.../upload/main/{path_in_repo}`
- It is not URL-encoded by this function.
- It should not start with `/` unless the caller deliberately wants a double slash in the URL.
- Nested paths may work if the server accepts them, but this function does not create directories explicitly.

### `content: bytes`

Raw file bytes sent as the request body.

Details:

- Sent with `Content-Type: application/octet-stream`.
- Empty `bytes` is allowed by the type and will attempt to upload an empty file.
- Large content is loaded into memory as a single request body.
- No compression, chunking, or Git LFS handling is performed here.

### `token: str`

HuggingFace access token used verbatim in the `Authorization` header.

Details:

- The caller must supply a valid token with write access.
- This function does **not** call `_resolve_token`.
- If `token` is empty, the header becomes `Authorization: Bearer ` and the server will likely respond with HTTP 401.
- If a caller violates the type and passes `None`, Python’s f-string will produce `Authorization: Bearer None`, which will also likely fail server-side.

## Return Value

Returns `None` on success.

Success means:

- `urlopen` completed without raising `HTTPError`
- `resp.read()` completed without raising
- the response was consumed and discarded

It does not return:

- the commit ID
- the uploaded URL
- the parsed repo ID
- any server response body

Callers must handle exceptions themselves.

## Algorithm

1. Normalize the repo ID:
   ```python
   parsed_id = resolve_repo_id(repo_id)
   ```

2. Build the upload URL:
   ```python
   url = f"{HF_BASE}/api/models/{parsed_id}/upload/main/{path_in_repo}"
   ```
   This hardcodes:
   - HuggingFace base URL
   - model repository API
   - `main` branch

3. Create a `urllib.request.Request`:
   - `data=content`
   - `method="PUT"`
   - `Authorization: Bearer {token}`
   - `Content-Type: application/octet-stream`

4. Execute the request:
   ```python
   with urlopen(req, timeout=60) as resp:
       resp.read()
   ```
   The response body is read to ensure the request completes and the response is closed.

5. On `HTTPError`:
   - If status is `401`, raise a `RuntimeError` with an authentication-specific message.
   - For any other HTTP status, raise a generic `RuntimeError` including `path_in_repo`, `parsed_id`, and the HTTP status code.

6. On success, fall through and return `None`.

## Side Effects

- Performs an external HTTP `PUT` request to HuggingFace.
- Creates or overwrites a remote file in the repository.
- Typically creates a new commit in the remote repository.
- Does not modify local files.
- Does not create the repository or branch.
- Does not cache or retain the token.

## Error Handling

### Converted to `RuntimeError`

- Any `HTTPError` with status `401`:
  - Message: authentication failed, suggests `huggingface-cli login` or `--token`.
- Any other `HTTPError`:
  - Message: `Failed to upload {path_in_repo} to {parsed_id}: HTTP {code}`.

Both are raised with `from e`, preserving the original HTTP error as the cause.

### Not converted

- `ValueError` from `resolve_repo_id` for malformed/non-HuggingFace URLs.
- `URLError` for network/DNS/connection failures.
- Timeout-related exceptions from `urlopen` or `resp.read()`.
- Potential URL/header construction errors from `urllib`.
- Errors raised while reading the response body.

So the function catches only `HTTPError`; it does not provide a uniform exception type for all failures.

## Usage Patterns

Typical usage:

```python
token = _resolve_token(args.token)
if not token:
    raise RuntimeError("No HuggingFace token available")

repo_id = create_repo("ddia-expert", token)
upload_file(
    repo_id=repo_id,
    path_in_repo="network.json",
    content=network_json.encode("utf-8"),
    token=token,
)
```

Caller obligations:

- Resolve or obtain a write token before calling.
- Ensure the repo exists, typically by calling `create_repo` first.
- Ensure `path_in_repo` is URL-safe or pre-encoded if it contains reserved characters.
- Handle overwrite semantics: uploading to an existing path replaces the remote file.
- Handle `RuntimeError` for HTTP failures and other exception types for transport/parse failures.
- Avoid assuming that an exception means the remote file was not changed.

## Dependencies

Internal dependencies:

- `HF_BASE` constant
- `resolve_repo_id` from `reasons/hf.py`

External/standard-library dependencies:

- `urllib.request.Request`
- `urllib.request.urlopen`
- `urllib.error.HTTPError`

Indirect dependency:

- `resolve_repo_id` reads `REASONS_HF_ORG` from the environment for bare repo names.

## Topics to Explore

- [function] `reasons/hf.py:resolve_repo_id` — Normalizes bare names, explicit `org/repo` IDs, and HuggingFace URLs before upload URL construction.
- [function] `reasons/hf.py:create_repo` — `upload_file` assumes the repo exists; this function creates model repos and handles the “already exists” case.
- [file] `tests/test_hf_default_org.py` — Covers default-org and URL-resolution behavior that `upload_file` indirectly relies on.
- [general] `huggingface-upload-api` — Understand the Hub single-file upload endpoint, branch/path semantics, size limits, and authentication behavior.
- [general] `urllib-error-handling` — Distinguish `HTTPError`, `URLError`, timeouts, and invalid-URL exceptions, since `upload_file` only wraps HTTP errors.

## Beliefs

- `upload-file-requires-existing-model-repo` — `upload_file` only issues a PUT to `/api/models/{parsed_id}/upload/main/{path_in_repo}` and never creates the repo or branch.
- `upload-file-does-not-resolve-token` — It uses the passed `token` directly and does not call `_resolve_token`, `HF_TOKEN`, or the cached token file.
- `upload-file-does-not-url-encode-path` — `path_in_repo` is interpolated directly into the URL, so reserved characters are not escaped by this function.
- `upload-file-wraps-only-http-errors` — Only `HTTPError` is converted to `RuntimeError`; `ValueError` and non-HTTP transport errors propagate.
- `upload-file-hardcodes-main-and-model-endpoint` — The upload URL always targets the `main` revision of the models API and cannot target datasets, spaces, or other branches.
