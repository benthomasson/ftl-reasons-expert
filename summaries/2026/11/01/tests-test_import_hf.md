---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_import_hf.py

# `tests/test_import_hf.py` — Explanation

## 1. Purpose

This file is the test suite for the **HuggingFace import pipeline** in the `ftl-reasons` project. Its responsibility is to verify that belief networks (a structured format of nodes, justifications, and nogoods) can be correctly downloaded from a HuggingFace model/data repository and loaded into a local SQLite database.

It covers the full stack of the import feature:
- **Parsing** a user-supplied repo identifier (bare ID or full URL)
- **Resolving** an authentication token from multiple sources
- **Downloading** the `network.json` file over HTTPS
- **Importing** the downloaded JSON into a local database via the `api` layer

By mocking `urlopen`, the tests run entirely offline — no network calls are made.

## 2. Key Components

### `SAMPLE_NETWORK` (module-level constant)
A minimal valid belief network JSON structure with two nodes (`a`, `b`), no nogoods, and meta containing `schema_version: "1.0"` and `project_name: "test-eem"`. Every test that exercises the download/import path returns this as the HTTP response body. It's the canonical "happy-path" fixture.

### `_mock_response(data)` (helper function)
Constructs a `MagicMock` that mimics a `urlopen` context-manager response:
- `.read()` returns the JSON-serialized body as UTF-8 bytes
- `__enter__` returns `self` (so `with urlopen(...) as resp:` works)
- `__exit__` returns `False`

This is the single point where all HTTP mocking is consolidated, keeping individual test methods clean.

### `TestParseRepoId`
Tests the `_parse_repo_id` function's contract:
| Input | Expected Output |
|-------|----------------|
| `"user/repo"` | `"user/repo"` (passthrough) |
| `"https://huggingface.co/user/repo"` | `"user/repo"` (strip scheme + host) |
| `"http://huggingface.co/user/repo"` | `"user/repo"` (http also accepted) |
| trailing slash | stripped |
| non-HF domain (`github.com`) | raises `ValueError("Not a HuggingFace URL")` |

### `TestResolveToken`
Tests the `_resolve_token` priority chain:
1. **Explicit argument** wins over everything
2. **`HF_TOKEN`** environment variable
3. **Cached file** at `~/.cache/huggingface/token`
4. **`None`** if nothing is found

Each test isolates one path using `monkeypatch.setenv` / `monkeypatch.delenv` and `patch("reasons.hf.Path.home")` to redirect the home-directory lookup into a temp directory.

### `TestDownloadNetwork`
Tests the `download_network` function:
- **Success path**: returns a JSON-serializable string of the network
- **URL construction**: asserts the exact `full_url` is `https://huggingface.co/{repo}/resolve/main/network.json`
- **Auth header**: `Authorization: Bearer {token}` present when a token is supplied
- **No auth header**: when `_resolve_token` returns `None`, the request has no `Authorization` header
- **401 → `RuntimeError("Authentication required")`**
- **404 → `RuntimeError("not found")`**
- **URL input**: passing a full URL still resolves to the same canonical path

### `TestImportHfApi`
Tests the high-level `api.import_hf` entry point:
- `init=True` creates the DB and returns a summary dict with `nodes_imported`, `nogoods_imported`, `repo_id`
- `init=False` (auto-init) still creates the DB and propagates `project_name` from the network's `meta`
- Import into an **existing** DB works (no `init` flag needed)
- `init=True` on an already-initialized DB does not fail
- Passing a full URL as `repo_id` still works end-to-end

## 3. Patterns

- **Class-based grouping by unit under test**: Each class (`TestParseRepoId`, `TestResolveToken`, `TestDownloadNetwork`, `TestImportHfApi`) maps 1:1 to a function or module-level concern. This makes it trivial to run a subset: `pytest tests/test_import_hf.py::TestResolveToken`.

- **`@patch` as method decorator**: Dependency injection at the `reasons.hf.urlopen` or `reasons.hf._resolve_token` level. The mock is injected into the module's namespace, not the caller's, ensuring the code under test sees the same reference.

- **`monkeypatch` for environment state**: Used for `HF_TOKEN` env var manipulation. This is pytest's preferred way to set/restore environment variables — it's automatically reverted after each test.

- **`tmp_path` for filesystem isolation**: Every DB-creation test uses `tmp_path` so tests are hermetic and can run in parallel.

- **Context-manager mocking**: `_mock_response` explicitly implements `__enter__`/`__exit__` because `urlopen` is used as `with urlopen(req) as resp:`. This is a non-obvious detail that would break tests if omitted.

- **Negative testing for HTTP errors**: `HTTPError` is raised via `side_effect` to simulate 401/404, and the test asserts both the exception type (`RuntimeError`) and the message pattern (`match="..."`).

## 4. Dependencies

### Imports (this file depends on)
| Dependency | Role |
|-----------|------|
| `reasons.hf._parse_repo_id` | Unit under test — repo ID normalization |
| `reasons.hf._resolve_token` | Unit under test — token resolution chain |
| `reasons.hf.download_network` | Unit under test — HTTP download + error mapping |
| `reasons.api` | Unit under test (integration level) — `import_hf`, `init_db`, `export_network` |
| `unittest.mock` | `MagicMock`, `patch` for HTTP and filesystem mocking |
| `urllib.error.HTTPError` | Simulated server error responses |
| `pytest` | Test runner, `monkeypatch`, `tmp_path` fixtures |

### Imported by
This is a **leaf test file** — nothing in the codebase imports it. It is discovered and executed by `pytest` via the `tests/` directory convention.

## 5. Flow

The test execution flow for the most complex test (`TestImportHfApi.test_import_with_init`) is:

```
1. pytest instantiates tmp_path fixture (isolated temp dir)
2. @patch("reasons.hf.urlopen") injects mock_urlopen
3. mock_urlopen.return_value = _mock_response(SAMPLE_NETWORK)
   → .read() will return JSON bytes of SAMPLE_NETWORK
4. api.import_hf("user/repo", init=True, token="tok", db_path=db)
   → internally calls _parse_repo_id("user/repo") → "user/repo"
   → internally calls _resolve_token("tok") → "tok"
   → internally calls download_network("user/repo", token="tok")
       → constructs Request("https://huggingface.co/user/repo/resolve/main/network.json")
       → sets Authorization: Bearer tok
       → calls urlopen(req)  ← mocked, returns _mock_response
       → with resp: resp.read() → JSON bytes
       → returns JSON string
   → parses JSON, creates DB at db_path, inserts nodes
   → returns {"nodes_imported": 2, "nogoods_imported": 0, "repo_id": "user/repo"}
5. Assertions verify the return dict and that the DB file exists
6. monkeypatch/patch decorators restore original state
```

## 6. Invariants

- **Repo ID canonical form**: After `_parse_repo_id`, the result is always `"{org}/{repo}"` with no scheme, no host, no trailing slash. The download URL is always `https://huggingface.co/{repo_id}/resolve/main/network.json`.
- **Token priority is strict**: explicit > `HF_TOKEN` env > `~/.cache/huggingface/token` file > `None`. An explicit token always shadows the env var (verified by `test_explicit_over_env`).
- **Auth header is all-or-nothing**: Either `Authorization: Bearer <token>` is present, or the header is entirely absent. There is no partial auth state.
- **`nodes_imported` equals the number of entries in the network's `nodes` dict**: 2 in SAMPLE_NETWORK → 2 reported.
- **`project_name` from network `meta` is propagated to the database** even in auto-init mode (`test_auto_init_sets_project_name`).

## 7. Error Handling

| Condition | Exception Raised | Message Pattern |
|-----------|-----------------|-----------------|
| Non-HuggingFace URL passed to `_parse_repo_id` | `ValueError` | `"Not a HuggingFace URL"` |
| HTTP 401 from `urlopen` | `RuntimeError` (from `download_network`) | `"Authentication required"` |
| HTTP 404 from `urlopen` | `RuntimeError` (from `download_network`) | `"not found"` |

The tests verify that `download_network` **translates** raw `HTTPError` exceptions (from `urllib`) into higher-level `RuntimeError` exceptions with actionable messages. Callers of `api.import_hf` never see `HTTPError` directly.

Notably, the tests do **not** cover:
- Network timeout / `URLError` (connection refused)
- Malformed JSON in the response body
- 500-series server errors
- Large payloads or streaming behavior

---

## Topics to Explore

- [file] `reasons/hf.py` — The implementation under test: how `_parse_repo_id`, `_resolve_token`, and `download_network` are actually coded, including the `urlopen` call site and error translation logic.
- [file] `reasons/api.py:import_hf` — The orchestration layer that ties download → parse → DB insert together; understand how `init=True` vs auto-init branches differ and what the return dict guarantees.
- [file] `tests/test_hf_default_org.py` — A sibling test file that likely covers the default-organization fallback behavior (e.g., when only a repo name is given without an org), complementing this file's coverage.
- [general] `belief-network-json-schema` — The `SAMPLE_NETWORK` structure (nodes, nogoods, repos, meta) is the wire format. Understanding the full schema (especially `justifications` and `repos` fields) is essential to knowing what the import must preserve.
- [file] `tests/conftest.py` — Shared fixtures (database setup, mock LLM, etc.) that may affect how `api.init_db` and `api.export_network` behave in this test context.

---

## Beliefs

- `import-hf-requires-hf-host` — `_parse_repo_id` raises `ValueError` for any URL whose host is not `huggingface.co`, so the download is hard-locked to that domain.
- `token-resolution-has-strict-priority` — `_resolve_token` follows the exact order: explicit arg → `HF_TOKEN` env → `~/.cache/huggingface/token` → `None`; an explicit token always shadows the environment variable.
- `download-network-translates-http-errors` — `download_network` converts `HTTPError(401)` to `RuntimeError("Authentication required…")` and `HTTPError(404)` to `RuntimeError("…not found…")`; callers never see raw `urllib` exceptions.
- `import-hf-produces-deterministic-count` — `api.import_hf` returns `nodes_imported` equal to `len(network["nodes"])` and `nogoods_imported` equal to `len(network["nogoods"])`, giving callers a verifiable count of what was persisted.
- `auto-init-propagates-project-name` — When `init=False` (auto-init), the `project_name` field from the downloaded network's `meta` is written into the database, making it recoverable via `api.export_network`.
