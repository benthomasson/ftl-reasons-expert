---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: create_repo in reasons/hf.py

# `create_repo` — Explanation

## Purpose
Provisions a HuggingFace **model** repository on the Hub before the caller uploads files to it. It centralizes ID normalization (`resolve_repo_id`), builds the small JSON body HuggingFace's `/api/repos/create` endpoint expects, and treats an "already exists" response as success so publishing flows can be re-run without special-casing. It exists mainly to make `upload_file` safe to call — the repo has to exist before files can be PUT into it.

## Contract
- **Preconditions**
  - `token` is a valid HuggingFace access token with write scope for the target org (not enforced).
  - `repo_id` is either a bare name, an `org/name` pair, or a `https://huggingface.co/...` URL (enforced by `resolve_repo_id`).
- **Postconditions on success**
  - The model repo `parsed_id` exists on the Hub with the requested `private` visibility *if it was just created*.
  - If it already existed, nothing was changed (visibility is NOT updated).
  - Returns the normalized `org/name` string.
- **Invariants**
  - Targets `f"{HF_BASE}/api/repos/create"` (i.e., `https://huggingface.co/api/repos/create`) with `method="POST"`, `Content-Type: application/json`, `Authorization: Bearer <token>`.
  - Payload always contains `{name, organization, private, type: "model"}`.
- **Not enforced but assumed**
  - `parsed_id` contains exactly one `/`. `split("/", 1)` silently allows nested names (`org = "a"`, `name = "b/c"`), which the Hub would likely reject.
  - `token` is non-empty. `Optional` typing isn't used, but an empty string just becomes `Authorization: Bearer ` and fails at the server with 401.

## Parameters
| Name | Type | Meaning | Edge cases |
|---|---|---|---|
| `repo_id` | `str` | Repo identifier. Bare name → `EEM-Hub/<name>` (or `REASONS_HF_ORG`); `org/name` → as-is; `https://huggingface.co/org/repo` → `org/repo`. | Whitespace and trailing `/` stripped. Non-HF URLs raise `ValueError` from `resolve_repo_id`. Names with extra slashes pass through and get mangled by `split("/", 1)`. |
| `token` | `str` | Bearer token for the Hub. | **Required** — unlike `download_network`, this function does NOT fall back to `HF_TOKEN` or `~/.cache/huggingface/token`. Callers must supply it. Empty string is syntactically accepted but fails as 401. |
| `private` | `bool = False` | Visibility for the newly created repo. | Ignored when repo already exists (409 path); the existing repo's visibility is untouched. |

## Return Value
Returns the resolved `org/name` string (`parsed_id`). This is the *normalized* identifier, not the raw `repo_id` argument, and not the server's response body (which is read and discarded). Callers can safely pass this return value on to `upload_file` since it re-normalizes anyway.

Callers must handle:
- Success path (either created or already existed) → any `str` return.
- `RuntimeError` from 401 or other HTTP statuses.
- `URLError` / `socket.timeout` — see Error Handling.

## Algorithm
1. `parsed_id = resolve_repo_id(repo_id)` — normalize (strip, URL handling, default org).
2. `org, name = parsed_id.split("/", 1)` — split on the *first* slash only.
3. Build and JSON-encode a body: `{"name": name, "organization": org, "private": private, "type": "model"}` (UTF-8 by default via `str.encode()`).
4. Construct `urllib.request.Request` with `POST`, JSON content type, Bearer auth.
5. `urlopen(req, timeout=30)` and `resp.read()` to drain the response (needed to close the connection cleanly).
6. On `HTTPError`:
   - `409` → `pass` (already exists).
   - `401` → `RuntimeError` with login hint.
   - anything else → `RuntimeError` with the HTTP code.
7. Return `parsed_id`.

## Side Effects
- **Network I/O**: single HTTPS `POST` to `huggingface.co/api/repos/create` with a 30s timeout.
- **State mutation on the Hub**: may create a new public/private model repo.
- **No local state changes**, no logging, no file writes, no env mutation.
- Reads the response body purely to drain/close it; the parsed response is not returned.

## Error Handling
| Condition | Behavior |
|---|---|
| `HTTPError 409 Conflict` | Swallowed → treated as success. |
| `HTTPError 401 Unauthorized` | `RuntimeError("Authentication failed. Run 'huggingface-cli login' or pass --token.")` chained from the original. |
| `HTTPError` any other code (403, 429, 500, …) | `RuntimeError(f"Failed to create repo {parsed_id}: HTTP {e.code}")`. |
| `urllib.error.URLError` (DNS, TLS, connection refused, timeout exceptions) | **Not caught** — propagates out of the function. This is the main gap versus `HTTPError` handling; callers that only catch `RuntimeError` will miss network failures. |
| `ValueError` from `resolve_repo_id` | Propagates (non-HF URL input). |
| Invalid JSON serialization | Not realistic here — payload values are primitives. |

Note that the 409 handling assumes the HuggingFace backend uses 409 exclusively for "repo already exists" — the code comments this assumption explicitly (`# repo already exists`).

## Usage Patterns
Typical publish flow:

```python
resolved = create_repo("ddia-expert", token=hf_token, private=True)
upload_file(resolved, "network.json", content, token=hf_token)
```

Caller obligations:
- Supply a real token explicitly (no env fallback in this function).
- Treat a successful return as "ready to upload" even if `private` didn't match the actual repo visibility.
- Catch `RuntimeError` *and* `URLError` if network failures matter.

## Dependencies
- **Internal**: `resolve_repo_id` (same module), `HF_BASE` constant, `DEFAULT_HF_ORG` indirectly via `resolve_repo_id`.
- **Stdlib**: `json`, `urllib.error.HTTPError`, `urllib.request.Request`, `urllib.request.urlopen`.
- **External service**: `https://huggingface.co/api/repos/create` (undocumented in-tree; contract inferred from HTTP codes).

## Topics to Explore
- [function] `reasons/hf.py:resolve_repo_id` — How IDs are normalized before this function sees them (default org, URL stripping, whitespace).
- [function] `reasons/hf.py:upload_file` — The companion operation this function is meant to precede; note it re-normalizes `repo_id`.
- [function] `reasons/hf.py:_resolve_token` — The token fallback logic used by `download_network` but deliberately *not* by `create_repo`.
- [file] `tests/test_hf_default_org.py` — Exercises the org-resolution logic that this function relies on.
- [general] `huggingface-repos-create-api` — The Hub's `POST /api/repos/create` semantics, especially its 409 and response-body shape, since this function discards the body.

## Beliefs
- `create-repo-is-idempotent` — A `409 Conflict` from the Hub is swallowed and the function returns the resolved ID as if the call succeeded.
- `create-repo-requires-explicit-token` — Unlike `download_network`, this function performs no `HF_TOKEN` / cached-token fallback; the token must be passed in.
- `create-repo-produces-resolved-repo-id` — The return value is `resolve_repo_id(repo_id)`, not the raw argument and not the server response.
- `create-repo-splits-on-first-slash` — The payload's `organization`/`name` derive from `parsed_id.split("/", 1)`, so extra slashes end up inside `name` rather than being rejected.
- `create-repo-does-not-handle-urlerror` — Non-`HTTPError` failures (DNS, TLS, timeout as `URLError`) propagate out of the function uncaught.
