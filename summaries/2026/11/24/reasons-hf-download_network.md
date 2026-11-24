---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: download_network in reasons/hf.py

## `download_network` — Explanation

### 1. Purpose

`download_network` fetches the file `network.json` from a HuggingFace hub repository on the `main` branch and returns its contents as a `str`. It is the read-side counterpart to `upload_file` in this module: the reasons pipeline uses it to pull a published belief network from `EEM-Hub/...` (or elsewhere) before parsing/loading it.

The function exists (rather than using `huggingface_hub`) to keep the module dependency-free — it only uses `urllib` — and to give the rest of the codebase a single place to normalize repo IDs, resolve auth, and translate HF HTTP status codes into friendlier errors.

### 2. Contract

**Preconditions**
- `repo_id` is a non-empty string. It may be a bare name (`ddia-expert`), `org/repo`, or a full `https://huggingface.co/org/repo[/*]` URL. Trailing whitespace and `/` are tolerated.
- The repo must exist and expose `network.json` on the `main` branch (no revision parameter exists).
- The caller will parse the returned string as JSON if that's their intent — this function does **not** validate it.

**Postconditions**
- Returns the raw UTF-8 body of the file as a string.
- The default HF org (`EEM-Hub`, overridable by `REASONS_HF_ORG`) is applied only for *bare* names, not for `org/repo` or URLs.

**Invariants**
- The URL path is always `{HF_BASE}/{parsed_id}/resolve/main/network.json`.
- Auth is only attached if a token is resolvable; otherwise the request is anonymous (public repos only).

### 3. Parameters

- **`repo_id: str`** — the repo to pull from. Normalized via `_parse_repo_id` → `resolve_repo_id`:
  - `"ddia-expert"` → `"EEM-Hub/ddia-expert"` (or `REASONS_HF_ORG` value if set).
  - `"myuser/my-eem"` → `"myuser/my-eem"` unchanged.
  - `"https://huggingface.co/org/repo"` → `"org/repo"`.
  - Trailing `/` and surrounding whitespace stripped.
  - **Edge case:** an `http(s)://` URL to a non-HF host → `ValueError("Not a HuggingFace URL: ...")` from `resolve_repo_id`.
  - **Type-system caveat:** empty string is not rejected; `resolve_repo_id` would prepend the default org, producing `EEM-Hub/`, and then the URL would 404.
- **`token: str | None = None`** — optional bearer token. If `None` (falsy), falls back to precedence: `HD_TOKEN`? No — falls back in `_resolve_token` to `HF_TOKEN` env var, then `~/.cache/huggingface/token` file (created by `huggingface-cli login`). An empty string is treated the same as `None` because `_resolve_token` uses `if token:`.

### 4. Return Value

- A `str` containing the UTF-8 decoded body of `network.json`, verbatim (may include a trailing newline; not stripped).
- No JSON deserialization, no schema checking, no ETag/cache headers honored.
- The caller is responsible for `json.loads` (and for handling `json.JSONDecodeError`).

### 5. Algorithm

1. `parsed_id = _parse_repo_id(repo_id)` — normalize to `[org/]name`.
2. Build URL: `https://huggingface.co/{parsed_id}/resolve/main/network.json`. (This is the standard HF "resolve" endpoint, which redirects to CDN storage; `urlopen` follows redirects by default.)
3. `resolved_token = _resolve_token(token)` — explicit → env → cached file.
4. If a token exists, set `headers["Authorization"] = "Bearer <token>"`. HF accepts either `Bearer` or `token` scheme; this uses `Bearer`.
5. Construct `Request(url, headers=headers)`.
6. `urlopen(req, timeout=30)` — HTTP GET, 30s socket timeout.
7. Read body, `.decode("utf-8")`, return.
8. On `HTTPError`:
   - `401` → raise `RuntimeError` telling the user to log in / pass `--token`.
   - `404` → raise `RuntimeError` with `{parsed_id}/network.json` for context.
   - any other status → bare `raise` (original `HTTPError` propagates).

### 6. Side Effects

- **Network I/O**: exactly one outbound HTTPS GET (plus redirects).
- **Env read**: `HF_TOKEN` (in `_resolve_token`) and `REASONS_HF_ORG` (in `resolve_repo_id`), both read at call time.
- **Filesystem read**: `~/.cache/huggingface/token` is read if `HF_TOKEN` is unset and no explicit token is passed.
- No mutation of global state, no caching, no writes.

### 7. Error Handling

The function has a deliberately narrow handler — only `HTTPError` is caught:

| Source | Exception surfaced |
|---|---|
| Malformed non-HF URL | `ValueError` (from `resolve_repo_id`) |
| HTTP 401 | `RuntimeError("Authentication required for …")` chained from `HTTPError` |
| HTTP 404 | `RuntimeError("Repository or file not found: …/network.json")` |
| Other HTTP (403, 500, 429, …) | Original `HTTPError` propagates |
| `URLError` (DNS, connection refused, TLS) | Propagates unwrapped |
| `TimeoutError` / `socket.timeout` | Propagates unwrapped |
| `UnicodeDecodeError` (non-UTF-8 body) | Propagates |
| `json.JSONDecodeError` | Never raised — body isn't parsed |

Notable gap: **private-repo access without permission commonly returns 403, not 401**, on HF. That falls through the `raise` and surfaces as a raw `HTTPError`, not the friendly auth message. Similarly, network-level failures skip the "run `huggingface-cli login`" hint even though they can also be caused by auth.

### 8. Usage Patterns

Typical call:

```python
raw = download_network("ddia-expert")           # default org
raw = download_network("someuser/my-net")       # explicit org
raw = download_network("https://huggingface.co/EEM-Hub/ddia-expert")
raw = download_network("private-net", token=os.environ["EXTRA_HF_TOKEN"])
network = json.loads(raw)
```

Caller obligations:
- Wrap in `try/except` if the repo might be missing or need auth — `RuntimeError` is the deliberate signal for those cases.
- Parse the returned text themselves; don't assume the file already exists on `main`.
- If depending on a non-default org, set `REASONS_HF_ORG` **before** importing/using, since `resolve_repo_id` reads it at call time (not at import).

### 9. Dependencies

- **Internal**: `resolve_repo_id` (and its alias `_parse_repo_id`), `_resolve_token`, constant `HF_BASE`, constant `DEFAULT_HF_ORG`.
- **Stdlib**: `urllib.request.{Request, urlopen}`, `urllib.error.HTTPError`, `os.environ`, `pathlib.Path` (via `_resolve_token`).
- **No third-party deps** — deliberately avoids `huggingface_hub`.

### Assumptions not enforced by type system

- `repo_id` is non-empty and is either a bare name, `org/repo`, or a real `huggingface.co` URL. A URL to any other host triggers `ValueError`, but an empty string produces a nonsensical URL rather than an error.
- A string starting with `/` is interpreted as `org/repo` (since `"/" in repo_id` matches) — so `"/my-repo"` becomes `"/my-repo"`, giving a malformed URL, not a default-org lookup.
- The file is UTF-8 JSON; the function neither validates this nor enforces a max payload size, so a huge (or HTML error-page) response is returned as-is.
- The target branch is always `main` — no `revision` parameter.

---

## Topics to Explore

- [function] `reasons/hf.py:resolve_repo_id` — Bare-name → default-org mapping and URL unwrapping that `download_network` inherits.
- [function] `reasons/hf.py:_resolve_token` — The explicit → `HF_TOKEN` → cached-file precedence that determines auth.
- [function] `reasons/hf.py:upload_file` — Symmetric write-only path; useful to see how `download_network`'s output is expected to round-trip.
- [general] `hf-403-vs-401` — Why real-world private-repo access often raises a raw `HTTPError` here instead of the friendly `RuntimeError`.
- [general] `network-json-schema` — What schema/shape the callers actually expect from the returned string.

## Beliefs

- `download-network-hardcodes-main-branch` — Every URL targets `resolve/main/network.json`; no revision argument exists.
- `resolve-token-precedence` — Auth is resolved from explicit token, then `HF_TOKEN`, then `~/.cache/huggingface/token`, in that order.
- `download-network-returns-unparsed-string` — The function returns UTF-8 text and never invokes `json.loads`.
- `http-401-and-404-map-to-runtimeerror` — Only those two status codes are rewritten; all other HTTP and network errors propagate unchanged.
- `download-network-requires-network-io` — The function performs exactly one `urlopen` GET (subject to HTTP redirects) with a 30-second timeout.
