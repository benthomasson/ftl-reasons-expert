---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/hf.py

# `reasons/hf.py` — HuggingFace Hub Integration for Belief Networks

## 1. Purpose

This file is the **HuggingFace I/O boundary** for the project. It owns the responsibility of moving belief network data (a `network.json` file) to and from HuggingFace repos. Specifically:

- **Download** a serialized belief network from an HF repo (`download_network`)
- **Create** an HF repo if it doesn't exist yet (`create_repo`)
- **Upload** a file into an HF repo (`upload_file`)

It does *not* define what a "belief network" is, how to serialize it, or what to do with it after download — those concerns live in `reasons/api.py` and the `network`/`storage` modules. This file is purely the transport + auth layer for the HF Hub.

Notably, it avoids the `huggingface_hub` SDK entirely and speaks raw HTTP via stdlib `urllib`. This keeps the dependency surface at zero external packages.

## 2. Key Components

| Component | Signature / Value | Contract |
|---|---|---|
| `HF_BASE` | `"https://huggingface.co"` | Single source of truth for the HF endpoint. Change this to point at a mirror or local server. |
| `DEFAULT_HF_ORG` | `"EEM-Hub"` | Fallback org when a user passes a bare repo name. Overridable via `REASONS_HF_ORG` env var. |
| `_resolve_token(token)` | `str \| None → str \| None` | Resolution chain: **explicit arg → `HF_TOKEN` env → `~/.cache/huggingface/token` file → `None`**. Returns `None` if no token is found anywhere. |
| `resolve_repo_id(repo_id)` | `str → str` | Normalizes any input form to `org/name`. Accepts bare names (`ddia-expert`), explicit orgs (`myuser/my-eem`), or full URLs (`https://huggingface.co/org/repo`). |
| `_parse_repo_id(repo_id)` | `str → str` | Thin alias that delegates to `resolve_repo_id`. Exists for readability at call sites. |
| `download_network(repo_id, token)` | `str, str\|None → str` | GETs `{HF_BASE}/{org}/{name}/resolve/main/network.json`. Returns the raw JSON **string** (caller is responsible for parsing). Token is optional. |
| `create_repo(repo_id, token, private)` | `str, str, bool → str` | POSTs to `/api/repos/create`. **Silently succeeds on 409** (repo already exists). Requires a valid token. |
| `upload_file(repo_id, path_in_repo, content, token)` | `str, str, bytes, str → None` | PUTs binary content to `/api/models/{org}/{name}/upload/main/{path}`. Requires a valid token. |

### Token Resolution Detail (`_resolve_token`)

```
1. If `token` arg is truthy  → return it
2. If `HF_TOKEN` env is set  → return it
3. If ~/.cache/huggingface/token exists → read & strip it
4. Otherwise                 → return None
```

This mirrors how `huggingface-cli` resolves credentials, so users who already ran `huggingface-cli login` will "just work" without passing `--token`.

## 3. Patterns & Conventions

- **Stdlib-only HTTP.** No `requests`, no `httpx`, no `huggingface_hub`. Every call goes through `urllib.request.Request` + `urlopen`. This is deliberate — the project's `pyproject.toml` can stay lean.
- **Functional, no classes.** Every public function is a top-level function. State is passed in (token, repo_id) rather than stored in an object. This makes the module trivially testable — you mock `urlopen` and go.
- **Error wrapping at the boundary.** Raw `HTTPError` is *never* leaked to callers. Each function catches it and re-raises a `RuntimeError` with an **actionable message** (e.g. "Run `huggingface-cli login`"). The original exception is chained via `from e`.
- **409-as-idempotency.** `create_repo` treats HTTP 409 (Conflict) as "already exists, no problem" — making the create-then-upload flow safe to retry.
- **`reasons/` vs `reasons_lib/`.** The repo contains both `reasons/hf.py` and `reasons_lib/hf.py`. The `reasons` package is the source; `reasons_lib` is a flat mirror (likely produced by a build/packaging step for distribution as a library). They should stay in lockstep.

## 4. Dependencies

### Imports (all stdlib)

```
json                          # for create_repo's payload
os                            # for env var lookups (HF_TOKEN, REASONS_HF_ORG)
pathlib.Path                  # for the cached token file path
urllib.error.HTTPError        # for error code inspection
urllib.request.Request, urlopen  # for HTTP calls
```

No third-party imports. No intra-project imports. This module is fully self-contained.

### Importers

| Importer | Likely usage |
|---|---|
| `reasons/api.py` | Calls `download_network` / `create_repo` / `upload_file` as part of the public API surface (import/export belief networks). |
| `reasons_lib/hf.py` | Mirror copy for library distribution. |
| `tests/test_hf_default_org.py` | Verifies `resolve_repo_id` prepends `EEM-Hub` for bare names and respects `REASONS_HF_ORG`. |
| `tests/test_import_hf.py` | End-to-end (or mocked) tests for the download → parse → import pipeline. |

## 5. Execution Flow

### Download path (most common)

```
caller: download_network("ddia-expert")
  │
  ├─ _parse_repo_id("ddia-expert")
  │    └─ resolve_repo_id: no "/" → prepend default org
  │       → "EEM-Hub/ddia-expert"
  │
  ├─ _resolve_token(None)
  │    └─ checks HF_TOKEN env, then ~/.cache/huggingface/token
  │
  ├─ builds URL: https://huggingface.co/EEM-Hub/ddia-expert/resolve/main/network.json
  │
  ├─ urlopen(Request(url, headers={Authorization: Bearer ...}), timeout=30)
  │
  ├─ on 200 → resp.read().decode("utf-8") → return JSON string
  │
  └─ on HTTPError:
       401 → RuntimeError("Authentication required …")
       404 → RuntimeError("Repository or file not found …")
       other → re-raise as-is
```

### Publish path (create + upload)

```
caller: create_repo("my-eem", token)
  │
  ├─ resolve_repo_id → "EEM-Hub/my-eem"
  ├─ POST /api/repos/create  {"name":"my-eem","organization":"EEM-Hub","private":false,"type":"model"}
  ├─ 201 → OK
  ├─ 409 → silently pass (idempotent)
  └─ 401 → RuntimeError("Authentication failed …")

caller: upload_file("my-eem", "network.json", json_bytes, token)
  │
  ├─ PUT /api/models/EEM-Hub/my-eem/upload/main/network.json
  ├─ 200 → None
  └─ 401 → RuntimeError("Authentication failed …")
```

## 6. Invariants

- **Repo ID is always `org/name` after resolution.** Every public function calls `resolve_repo_id` (or `_parse_repo_id`) before touching a URL. You will never see a bare name in an actual HTTP request.
- **The downloaded file is always `network.json` on the `main` branch.** This is hardcoded in `download_network`. There is no parameter for branch or filename.
- **Uploads are typed as "model" repos.** `create_repo` hardcodes `"type": "model"`. The upload URL uses `/api/models/…`. If you ever need a dataset or space, this file won't work as-is.
- **`upload_file` and `create_repo` require a non-`None` token.** They don't call `_resolve_token` internally — the caller is responsible for passing a valid token. (Contrast with `download_network`, which tolerates `None`.)
- **Timeouts differ by operation.** Download: 30 s. Create: 30 s. Upload: 60 s. The longer upload timeout accounts for potentially large `network.json` payloads.

## 7. Error Handling

| HTTP code | Where | Behavior |
|---|---|---|
| **401** | `download_network`, `create_repo`, `upload_file` | Wrapped in `RuntimeError` with the hint: *"Run `huggingface-cli login` or pass `--token`."* Original `HTTPError` chained. |
| **404** | `download_network` only | `RuntimeError`: *"Repository or file not found: {id}/network.json."* |
| **409** | `create_repo` only | **Swallowed silently** (`pass`). Treats "repo already exists" as success. |
| **Other** | All three | `download_network`: re-raised as-is. `create_repo` / `upload_file`: wrapped in `RuntimeError` with the HTTP code. |
| **Non-HTTP exceptions** (DNS, timeout, connection reset) | All three | Propagate unhandled. The caller (e.g. `reasons/api.py`) is expected to catch these. |

The design intent: **the caller never sees a raw `HTTPError`**. Every error that reaches the caller is a `RuntimeError` with a human-readable, actionable message. The only exception is `download_network`'s fallthrough for non-401/404 codes, which re-raises the `HTTPError` directly — a minor inconsistency worth noting.

---

## Topics to Explore

- [file] `reasons/api.py` — The consumer of every function in this module; shows how `download_network`'s return value (a JSON string) is parsed and fed into the belief network, and how the publish path sequences `create_repo` → `upload_file`.
- [file] `tests/test_hf_default_org.py` — Locks in the `resolve_repo_id` contract: bare names get `EEM-Hub` prepended, explicit orgs pass through, and `REASONS_HF_ORG` overrides the default. Read this to understand the exact normalization rules before touching `resolve_repo_id`.
- [file] `reasons/network.py` — Defines the belief network data model that `network.json` serializes. Understanding the schema makes the "download → parse → import" pipeline in `api.py` concrete.
- [general] `reasons-vs-reasons-lib-duplication` — The repo ships both `reasons/` and `reasons_lib/` with mirrored files. Understanding the build/packaging step that generates `reasons_lib` from `reasons` (check `pyproject.toml`, `.forge/`) is critical before editing either copy.
- [function] `reasons/hf.py:upload_file` — Uses the HF Hub's single-file PUT upload API. Explore the [HF Hub API docs](https://huggingface.co/docs/hub/api) to understand rate limits, LFS chunking, and whether a migration to `huggingface_hub.hf_hub_download` / `upload_file` would be warranted for larger payloads.

## Beliefs

- `download-network-returns-raw-json-string` — `download_network` returns the HTTP response body as a decoded `str`, not a `dict`; the caller (e.g. `reasons/api.py`) is responsible for `json.loads`.
- `create-repo-treats-409-as-success` — A 409 Conflict from the HF create-repo endpoint is swallowed via `pass`, making `create_repo` idempotent and safe to call before every upload.
- `hf-module-stdlib-only` — `reasons/hf.py` has zero third-party imports; all HTTP is done via `urllib.request`, so adding this module to a dependency-free bundle introduces no new packages.
- `upload-and-create-require-caller-supplied-token` — Unlike `download_network` (which resolves the token internally via `_resolve_token`), both `create_repo` and `upload_file` expect the caller to pass a non-`None` token; calling them with `None` produces a 401 at runtime, not a local guard.
- `reasons-extends-reasons-lib-mirror` — `reasons_lib/hf.py` is a flat mirror of `reasons/hf.py` for library distribution; edits must be applied to both (or the build step must be understood) to keep them in sync.
