---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_import_hf.py

# `tests/test_import_hf.py` — HuggingFace Import Test Suite

## Purpose

This file is the unit/integration test suite for the **HuggingFace belief-network import** feature. It validates the behavior of the `reasons.hf` module (repo-id normalization, token resolution, HTTP download) and the `reasons.api.import_hf` entry point (writing a downloaded network into a SQLite database).

It owns *one responsibility*: verifying that a user can point at either a bare `user/repo` identifier or a full `https://huggingface.co/...` URL and reliably import a `network.json` snapshot into a local DB — including correct URL construction, header/credential handling, and HTTP error mapping. It does **not** hit the network; every HTTP interaction is mocked.

---

## Key Components

### Constant: `SAMPLE_NETWORK`
```python
SAMPLE_NETWORK = {
    "meta": {"schema_version": "1.0", "project_name": "test-eem"},
    "nodes": { "a": {...}, "b": {...} },
    "nogoods": [],
    "repos": {},
}
```
A minimal two-node belief network. It is the canonical fixture payload returned by every mocked `urlopen` call. Two properties matter for assertions downstream:
- `meta.project_name == "test-eem"` — used to assert auto-init sets the project name from the network (not from a CLI default).
- Exactly 2 nodes and 0 nogoods — pinned by `nodes_imported == 2` / `nogoods_imported == 0` assertions.

### Helper: `_mock_response(data)`
```python
def _mock_response(data):
    body = json.dumps(data).encode("utf-8")
    resp = MagicMock()
    resp.read.return_value = body
    resp.__enter__ = lambda s: s
    resp.__exit__ = MagicMock(return_value=False)
    return resp
```
Builds a duck-typed stand-in for the object returned by `urllib.request.urlopen`. Two contracts it satisfies:
1. `.read()` returns the raw UTF-8 JSON bytes (so `download_network` can decode to `str`).
2. It is a **context manager** — `__enter__` returns itself, `__exit__` returns `False` (doesn't swallow exceptions). This mirrors `urlopen`'s actual protocol when used as `with urlopen(req) as resp:`.

If `download_network` ever changes to call `.status`, `.headers`, or iterate bytes, this helper must be extended.

### `TestParseRepoId`
Unit tests for `_parse_repo_id`. Contract revealed:
| Input | Output |
|---|---|
| `"user/repo"` | `"user/repo"` |
| `"https://huggingface.co/user/repo"` | `"user/repo"` |
| `"http://huggingface.co/user/repo"` | `"user/repo"` |
| `"https://huggingface.co/user/repo/"` | `"user/repo"` (trailing `/` stripped) |
| `"https://github.com/user/repo"` | raises `ValueError` matching `"Not a HuggingFace URL"` |

This enforces that only `huggingface.co` URLs (http or https) are accepted, and that a bare `user/repo` is passed through unchanged.

### `TestResolveToken`
Unit tests for `_resolve_token`. This documents a strict **precedence chain**:
1. Explicit argument (`_resolve_token("my-token") == "my-token"`).
2. `HF_TOKEN` env var.
3. Cached token file at `~/.cache/huggingface/token`.
4. `None` if none of the above.

Notable mechanics:
- `test_cached_file` writes `"file-token\n"` and expects `"file-token"` → the implementation **strips whitespace** from the file contents.
- `Path.home()` is patched via `reasons.hf.Path.home`, meaning the module imports `Path` directly and calls `Path.home()` — tests rely on that exact symbol path.
- `test_cached_file` explicitly `delenv("HF_TOKEN")` so a developer's own token doesn't leak into the test.
- `test_no_token_returns_none` patches `Path.home` to `/nonexistent` to guarantee the cached-file lookup misses.

### `TestDownloadNetwork`
Tests `download_network(repo_id_or_url, token=...)`, which returns the downloaded JSON **as a string** (test does `json.loads(result)`, not `result["nodes"]`). Behavior pinned:

- **Success path**: returns decodable JSON string containing the sample payload.
- **URL construction**: `https://huggingface.co/{repo}/resolve/main/network.json` — the `resolve/main/network.json` subpath is fixed by the test (`test_correct_url`). URL input is normalized first, then the same suffix is appended (`test_url_input`).
- **Auth header**: when a token is available, the request has `Authorization: Bearer <token>` (`test_auth_header_present`, checking `req.get_header("Authorization")` — so `download_network` must build a `urllib.request.Request`).
- **No token → no header**: `test_no_auth_header_without_token` patches `_resolve_token` to return `None` and asserts `req.has_header("Authorization")` is falsy.
- **HTTP error mapping** (`test_401_*`, `test_404_*`):
  - `401` → `RuntimeError` whose message matches `"Authentication required"`.
  - `404` → `RuntimeError` whose message matches `"not found"`.
  Both are produced by catching `HTTPError` from `urlopen`. Because `mock_urlopen.call_args[0][0]` retrieves the `Request` object passed positionally, the implementation must call `urlopen(req)` positionally (not as a keyword).

### `TestImportHfApi`
Integration-level tests for `api.import_hf(...)`, exercising a **real** SQLite file in `tmp_path` but with `urlopen` still mocked.

Signature inferred: `api.import_hf(repo_id_or_url, *, init=bool, token=str|None, db_path=str) -> dict`.

Return dict fields asserted:
- `nodes_imported` (int; `2` for `SAMPLE_NETWORK`)
- `nogoods_imported` (int; `0` for `SAMPLE_NETWORK`)
- `repo_id` (normalized `"user/repo"` — even when a full URL was passed)

Scenarios:
| Test | What it covers |
|---|---|
| `test_import_with_init` | Fresh DB + `init=True`; asserts file exists and counts match. |
| `test_auto_init_sets_project_name` | `init=False` on a nonexistent DB still creates it and copies `meta.project_name` from the downloaded network. Read back via `api.export_network`. |
| `test_import_into_existing_db` | Pre-existing DB via `api.init_db`; import works, no re-init needed. |
| `test_init_flag_with_existing_db` | `init=True` on an already-initialized DB must not fail (idempotent). |
| `test_import_hf_with_url` | URL input accepted; `repo_id` in result is normalized. |

---

## Patterns

- **`unittest.mock.patch` as a decorator** (`@patch("reasons.hf.urlopen")`): mocks the symbol at the *consumer* module (`reasons.hf`), not at `urllib.request`. This is the correct patching target — if the implementation switched to `requests` or `httpx`, these tests would break, which is intentional coupling to the chosen HTTP client.
- **Patch target ordering for stacked decorators**: when stacked, mocks are injected bottom-up as arguments. Example:
  ```python
  @patch("reasons.hf._resolve_token", return_value=None)   # outermost → last arg
  @patch("reasons.hf.urlopen")                              # innermost  → first arg
  def test_no_auth_header_without_token(self, mock_urlopen, mock_resolve):
  ```
  The parameter order `(self, mock_urlopen, mock_resolve)` matches the inner-to-outer decorator order.
- **Duck-typing a context manager** via `MagicMock.__enter__/__exit__` rather than using `unittest.mock.mock_open` or `httpx.MockTransport` — minimal and explicit.
- **Hermetic file-system tests**: `tmp_path` (pytest built-in) for DB files, and patched `Path.home` for the token cache — no writes outside `tmp_path`.
- **`monkeypatch` for env isolation**: every env-sensitive test either sets or (`delenv`) clears `HF_TOKEN` so results don't depend on the developer's shell.
- **Fixture payload builder** (`_mock_response`) is shared across suites, centralizing response-shape assumptions.

---

## Dependencies

### External imports (used)
- `json` — serializing `SAMPLE_NETWORK` and deserializing the download result.
- `io.BytesIO` — constructing the empty body for `HTTPError` instances.
- `pathlib.Path` — checking `Path(db).exists()` and using `Path("/nonexistent")` as a patched home.
- `unittest.mock.{MagicMock, patch}` — response stubbing and patching.
- `urllib.error.HTTPError` — simulating 401/404.
- `pytest` — `raises`, fixtures (`tmp_path`, `monkeypatch`).
- `reasons.api` — `import_hf`, `init_db`, `export_network`.
- `reasons.hf` — `_parse_repo_id`, `_resolve_token`, `download_network` (private helpers tested directly).

### External imports (unused)
- `os` is imported but never referenced. Harmless leftovers; safe to remove.

### What depends on this file
Nothing in production imports it. It is collected only by pytest. The `reasons_lib/` mirror directory does **not** have its own parallel test file here — these tests target `reasons.*`, so `reasons_lib` parity is untested by this file.

### Test-suite dependencies
Relies on `tests/conftest.py` for any shared fixtures (not visible here, but likely no special fixtures are needed since all state is local).

---

## Flow

### `download_network` (inferred from tests)
1. Normalize input: if it's a URL, `_parse_repo_id(...)` → `"user/repo"` (strips scheme, trailing `/`, validates host).
2. Resolve credentials: `token` arg → else `_resolve_token()` (env → cache file → `None`).
3. Build `urllib.request.Request(f"https://huggingface.co/{repo_id}/resolve/main/network.json")` and, if a token exists, `.add_header("Authorization", f"Bearer {token}")`.
4. `with urlopen(req) as resp:` → `resp.read()` → decode UTF-8 → return the raw string.
5. On `HTTPError`:
   - `code == 401` → `RuntimeError("Authentication required ...")`.
   - `code == 404` → `RuntimeError("... not found ...")`.
   - Other codes: not covered by this file — likely fall through or wrapped generically.

### `api.import_hf` (inferred)
1. Call `download_network(repo_id_or_url, token=token)` → JSON string.
2. `json.loads(...)` into a dict.
3. Ensure target DB exists:
   - If `init=True` → call init (idempotent — `test_init_flag_with_existing_db` proves it tolerates an existing DB).
   - Else if DB missing → auto-initialize and seed `meta.project_name` from downloaded `meta` (`test_auto_init_sets_project_name`).
   - Else use existing DB.
4. Import `nodes` and `nogoods`; count them.
5. Return `{"nodes_imported": ..., "nogoods_imported": ..., "repo_id": <normalized>}`.

### Per-test control flow
Every `TestDownloadNetwork` / `TestImportHfApi` test replaces `reasons.hf.urlopen` with a `MagicMock` whose return value (or `side_effect`) is a `_mock_response` or `HTTPError`. When `api.import_hf` calls `download_network`, that module-level `urlopen` is the mock — so the entire import path runs without network I/O, but the SQLite code executes for real.

---

## Invariants

- **URL suffix is fixed**: `network.json` is always fetched from `<repo>/resolve/main/network.json`. There is no branch for `dev`/tags in these tests.
- **Repos must be on `huggingface.co`**: `_parse_repo_id` raises `ValueError` for anything else. `github.com/...` is explicitly rejected.
- **Token precedence is strict**: explicit > `HF_TOKEN` env > `~/.cache/huggingface/token` > `None`. Tests lock each boundary.
- **Cached token file is whitespace-trimmed**: `"file-token\n"` → `"file-token"`.
- **Authorization header only when a token resolves**: absent token → no `Authorization` header (not even `Bearer `).
- **`repo_id` in the result dict is always normalized** to `user/repo`, even if the caller passed a full URL.
- **`import_hf` never leaves the DB in a non-existent state**: both `init=True` and auto-init (`init=False` on a missing DB) produce a working DB.
- **`import_hf` is idempotent under `init=True`**: passing `init=True` to an already-initialized DB must not raise.
- **Node/nogood counts come from the network payload**, not from a fixed default — `SAMPLE_NETWORK`'s 2/0 are echoed exactly.
- **HTTP client contract**: the implementation must call `urlopen(req)` with a positional `Request`, because tests read `mock_urlopen.call_args[0][0]` (positional-indexed).

---

## Error Handling

- **`ValueError` from `_parse_repo_id`**: raised synchronously for non-HF URLs, before any HTTP call. Tested via `pytest.raises(ValueError, match="Not a HuggingFace URL")`.
- **`urllib.error.HTTPError` → `RuntimeError`** inside `download_network`:
  - `401` → message includes `"Authentication required"` (guides the user to pass `--token` / set `HF_TOKEN`).
  - `404` → message includes `"not found"` (indicates wrong repo or missing `network.json`).
  - Any other `HTTPError` status is not covered by this file — a gap the maintainer should verify separately.
- **No swallowing of non-HTTP errors**: `test_correct_url` and others only inspect the request, so anything else (e.g. connection errors) would propagate — but those aren't tested here.
- **`_resolve_token` returns `None` rather than raising** when no credentials are found (test `test_no_token_returns_none`), letting the public download path decide whether anonymous access is acceptable.

---

## Topics to Explore

- [file] `reasons/hf.py` — The production implementation this suite tests; confirm the `401`/`404` mapping and whether other status codes are handled.
- [function] `reasons/api.py:import_hf` — Verify the auto-init / project-name seeding path that `test_auto_init_sets_project_name` exercises.
- [file] `tests/conftest.py` — Check for shared fixtures or HTTP patches that might interact with `@patch("reasons.hf.urlopen")`.
- [general] `hf-token-resolution-order` — Understand why the cache-file location is hardcoded to `~/.cache/huggingface/token` and how it aligns with the official `huggingface_hub` behavior.
- [file] `tests/test_hf_default_org.py` — Adjacent HF test module; likely covers the default organization fallback not exercised here.

---

## Beliefs

- `parse-repo-id-rejects-non-hf-hosts` — `_parse_repo_id` raises `ValueError("Not a HuggingFace URL ...")` for any URL whose host is not `huggingface.co`.
- `resolve-token-precedence-explicit-over-env-over-cache` — `_resolve_token` prefers the explicit argument, then `HF_TOKEN`, then `~/.cache/huggingface/token`, then `None`, in that order.
- `download-network-uses-fixed-network-json-path` — `download_network` always requests `<repo>/resolve/main/network.json` and appends it to the normalized repo id.
- `download-network-maps-401-and-404-to-runtime-error` — `HTTPError` with code 401 becomes a `RuntimeError` matching `"Authentication required"`; code 404 becomes a `RuntimeError` matching `"not found"`.
- `import-hf-normalizes-repo-id-in-result` — `api.import_hf` returns `repo_id` as `"user/repo"` regardless of whether the caller supplied a bare id or a full HuggingFace URL.
- `import-hf-auto-initializes-missing-db` — With `init=False` and a nonexistent `db_path`, `import_hf` creates the DB and seeds `meta.project_name` from the downloaded network's `meta.project_name`.
