---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/hf.py

# `reasons/hf.py` — HuggingFace Network Transport

## Purpose

This module is the project's **thin, dependency-free client for HuggingFace Hub**. It is responsible for three concrete operations:

1. **Download** a `network.json` file from a HF repo (`download_network`).
2. **Create** a HF repo if one does not exist (`create_repo`).
3. **Upload** a file to a HF repo (`upload_file`).

It exists so the rest of the codebase (e.g., `reasons/api.py`, `reasons/import_hf`) can round-trip belief networks through HuggingFace without pulling in the `huggingface_hub` SDK. It talks directly to the HTTP REST API using `urllib.request`.

## Key Components

### Constants
- `HF_BASE = "https://huggingface.co"` — the single host used for all requests.
- `DEFAULT_HF_ORG = "EEM-Hub"` — the fallback organization for bare repo names.

### Functions

**`_resolve_token(token: str | None = None) -> str | None`**
Resolves an auth token using a strict precedence chain:
1. The explicit `token` argument.
2. The `HF_TOKEN` environment variable.
3. The cached file at `~/.cache/huggingface/token` (the file `huggingface-cli login` writes).
4. Otherwise `None` (unauthenticated).

**`resolve_repo_id(repo_id: str) -> str`**
Normalizes any of these into `org/name`:
- `"ddia-expert"` → `"EEM-Hub/ddia-expert"` (default org)
- `"myuser/my-eem"` → `"myuser/my-eem"` (returned as-is)
- `"https://huggingface.co/org/repo"` → `"org/repo"`
- The default org is taken from `REASONS_HF_ORG` if set, else `DEFAULT_HF_ORG`.

This is the single normalization point; all other functions call it (or `_parse_repo_id`).

**`_parse_repo_id(repo_id: str) -> str`**
Legacy alias that just delegates to `resolve_repo_id`. Presumably kept for backwards-compatibility with earlier callers.

**`download_network(repo_id: str, token: str | None = None) -> str`**
- Resolves the repo, builds `GET {HF_BASE}/{parsed_id}/resolve/main/network.json`.
- Resolves token via `_resolve_token` and adds a `Bearer` header if present.
- Returns the decoded JSON **string** (not a parsed dict — callers parse it).
- 30 s timeout.

**`create_repo(repo_id: str, token: str, private: bool = False) -> str`**
- `POST {HF_BASE}/api/repos/create` with a JSON body `{"name", "organization", "private", "type": "model"}`.
- Returns the resolved `org/name`.
- Token is **required** here (no `_resolve_token` fallback).

**`upload_file(repo_id: str, path_in_repo: str, content: bytes, token: str) -> None`**
- `PUT {HF_BASE}/api/models/{parsed_id}/upload/main/{path_in_repo}` with the raw bytes.
- Token is **required**; no fallback.
- 60 s timeout.

## Patterns

- **Interpreter façade over a REST API**: three small functions map 1:1 to HTTP verbs + endpoints. No SDK, no session object, no gRPC.
- **Layered defaults chain** for the token and for the org: explicit argument → environment variable → on-disk cache / hard-coded default. This makes the module work both interactively (logged-in CLI) and in CI (env var).
- **Error translation at the boundary**: raw `HTTPError` is converted into a user-facing `RuntimeError` with a "run `huggingface-cli login`" hint, keeping the internal code free of HTTP status awareness.
- **Idempotency via status code**: `create_repo` treats `409 Conflict` as success.
- **Mirrored package**: `reasons_lib/hf.py` is a duplicate of this file; the repo ships two packages side-by-side (visible in repo context and `imported_by`).

## Dependencies

**Depends on (stdlib only):**
- `json` — payload encoding for `create_repo`.
- `os` — `HF_TOKEN` and `REASONS_HF_ORG` env lookups.
- `pathlib.Path` — locating `~/.cache/huggingface/token`.
- `urllib.error.HTTPError` — status-code handling.
- `urllib.request.Request`, `urlopen` — the actual HTTP layer.

No `requests`, no `huggingface_hub`.

**Depended on by:**
- `reasons/api.py` — likely the public `reasons import hf …` / `reasons export hf …` command surface.
- `reasons_lib/hf.py` — the parallel package copy.
- `tests/test_hf_default_org.py` — verifies `REASONS_HF_ORG` / `DEFAULT_HF_ORG` behavior.
- `tests/test_import_hf.py` — end-to-end import path from a HF repo.

## Flow

Typical publish flow (as wired via `reasons/api.py`):

1. Caller passes something like `"ddia-expert"` or a full URL.
2. `resolve_repo_id` → `"EEM-Hub/ddia-expert"`.
3. `_resolve_token()` returns a token from arg/env/cache.
4. `create_repo(...)` — 409 means already exists; continue.
5. `upload_file(..., "network.json", json_bytes, token)`.

Typical download flow:

1. Caller passes `"org/repo"` or a URL.
2. `resolve_repo_id` normalizes → `org/repo`.
3. URL is `https://huggingface.co/org/repo/resolve/main/network.json`.
4. Optional `Authorization: Bearer <token>` is set.
5. Response body is returned verbatim as a `str`.

## Invariants

- **Return shape of `resolve_repo_id` is always `"org/name"`.** Exactly one slash after normalization. Bare names are namespaced; explicit `org/name` is preserved.
- **Non-HuggingFace URLs are rejected.** A URL that doesn't contain `huggingface.co/` raises `ValueError` rather than silently mis-parsing.
- **`create_repo` and `upload_file` require an explicit token.** Only `download_network` goes through `_resolve_token`.
- **`create_repo` is idempotent** w.r.t. an existing repo (409 is swallowed).
- **`download_network` returns unparsed JSON text**, so upstream code owns schema validation.
- **Endpoints are hard-wired to the `main` revision** (`/resolve/main/…` and `/upload/main/…`). There is no branch/revision parameter today — a caller wanting a different branch must extend the signature.

## Error Handling

Different functions apply different policies — worth calling out explicitly:

| Situation | `download_network` | `create_repo` / `upload_file` |
|---|---|---|
| 401 | `RuntimeError("Authentication required … run 'huggingface-cli login'")` | `RuntimeError("Authentication failed …")` |
| 404 | `RuntimeError("Repository or file not found: …")` | generic `RuntimeError("Failed to … HTTP 404")` |
| 409 | — | swallowed (repo exists) |
| other HTTPError | **re-raised raw** (the `raise` on the last line of the `except`) | wrapped in `RuntimeError` with the status code |
| `ValueError` on bad URL | propagates | propagates |
| network / timeout / `URLError` | **not caught** — propagates | **not caught** — propagates |

Two consequences a maintainer should know:

- `download_network`'s non-401/404 branch does a bare `raise` — so callers must be prepared to catch `HTTPError` (e.g., 5xx) in addition to `RuntimeError`. This is an inconsistency with the other two functions.
- Timeouts on unauthenticated public repos and rate limits (429) are not translated; they surface as raw `HTTPError` for download and wrapped `RuntimeError` for upload.

## Topics to Explore

- [file] `reasons/api.py` — how `hf.py` is wired into the `import hf` / `export hf` CLI commands and what JSON schema it expects from `network.json`.
- [function] `reasons/hf.py:resolve_repo_id` — exercise the three input shapes (bare name, `org/name`, URL) plus the `REASONS_HF_ORG` override to confirm the normalization behavior.
- [file] `tests/test_import_hf.py` — shows the end-to-end expected contract for `download_network` and how responses are mocked.
- [file] `tests/test_hf_default_org.py` — verifies the default-org and env-override rules, which are the most subtle part of `resolve_repo_id`.
- [general] `reasons-vs-reasons_lib-mirror` — the repo ships two packages with near-identical files; investigate how they stay in sync (or don't).

## Beliefs

- `hf-uses-stdlib-urllib-not-sdk` — The module deliberately uses `urllib.request` and stdlib only; there is no `huggingface_hub` import.
- `hf-token-resolution-order` — `_resolve_token` resolves in the order: explicit arg → `HF_TOKEN` env → `~/.cache/huggingface/token`.
- `default-org-is-eem-hub` — Bare repo names are namespaced under `EEM-Hub` unless `REASONS_HF_ORG` overrides it.
- `resolve-repo-id-always-returns-org-name` — `resolve_repo_id` always returns a single-slash `org/name` string and raises `ValueError` for non-HuggingFace URLs.
- `create-repo-swallows-409` — `create_repo` treats HTTP 409 as success so it can be called unconditionally before an upload.
- `download-network-returns-raw-json-text` — `download_network` returns the response body as a `str`; parsing is the caller's responsibility.
- `download-network-rethrows-non-401-404` — `download_network` only wraps 401 and 404 into `RuntimeError`; other `HTTPError` codes propagate raw, unlike `create_repo` and `upload_file` which wrap all codes.
