---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_hf_default_org.py

# `tests/test_hf_default_org.py`

## Purpose

This file is the integration test suite for the HuggingFace (HF) publishing and pulling layer in the `ftl-reasons` project. It verifies that the `reasons.hf` module correctly:

- Resolves bare repository names to the `EEM-Hub` default organization
- Creates HF model repos via the HF API
- Uploads files to an existing repo
- Orchestrates a full `publish_hf` flow (create repo + upload 3 files)
- Orchestrates a full `import_hf` (pull) flow
- Exposes the correct CLI dispatch handlers (`cmd_pull`, `cmd_publish`)

It sits alongside `test_import_hf.py` (which likely tests the import path in isolation) and specifically guards the **default-org resolution** behavior and the **publish** path that was added on top of the existing import.

## Key Components

| Component | Contract |
|-----------|----------|
| `SAMPLE_NETWORK` | A minimal valid network JSON payload (1 node, empty nogoods/repos) used as mock HTTP response body for pull tests. |
| `_mock_response(data=None)` | Returns a `MagicMock` that mimics `urllib.request.urlopen`'s context-manager protocol: `.read()` yields JSON-encoded bytes, `__enter__`/`__exit__` make it usable with `with`. |
| `TestResolveRepoId` | Verifies `resolve_repo_id(repo_id)` — a pure string transformer. Bare names → `EEM-Hub/<name>`. `org/name` passthrough. URL parsing. Env-var override via `REASONS_HF_ORG`. Non-HF URLs raise `ValueError`. |
| `TestCreateRepo` | Verifies `create_repo(name, token, private)` issues a `POST` to `https://huggingface.co/api/repos/create` with the correct JSON body. 409 → idempotent no-op. 401 → `RuntimeError("Authentication failed…")`. Other HTTP errors → `RuntimeError("HTTP <code>…")`. |
| `TestUploadFile` | Verifies `upload_file(repo_id, filename, data, token)` issues a `PUT` to `…/upload/main/<filename>` with `Authorization: Bearer <token>`. Error mapping same as `create_repo`. |
| `TestPublishHf` | Verifies `api.publish_hf(repo_id, token, db_path, private)` returns `{"repo_id", "url", "files_uploaded"}` where `files_uploaded == {"network.json", "beliefs.md", "README.md"}`. Asserts exactly 4 `urlopen` calls (1 create + 3 uploads). Requires a token; raises `RuntimeError` if none found. |
| `TestPullCommand` | Verifies `api.import_hf(repo_id, init=True, token, db_path)` resolves the repo, fetches `network.json`, deserializes it, and imports nodes. |
| `TestCLIDispatch` | Smoke-tests that `cli.cmd_pull`, `cli.cmd_import_hf`, and `cli.cmd_publish` are importable callables. |

The `db` fixture creates a temporary SQLite database with one node (`n1`) so that `publish_hf` has content to serialize.

## Patterns

1. **HTTP boundary mocking via `@patch("reasons.hf.urlopen")`** — Every test that touches the network patches the single `urlopen` call inside `reasons.hf`. This isolates the test from real HF and lets assertions inspect `mock_urlopen.call_args` to verify URL, method, headers, and body. No HTTP server, no requests-mock library.

2. **Context-manager protocol on the mock response** — `_mock_response` wires up `__enter__`/`__exit__` so code under test can do `with urlopen(req) as resp: resp.read()`. This is a common idiom when the production code uses `urlopen` as a context manager (which it does, based on the test structure).

3. **Class-based grouping by SUT** — Each `Test*` class maps 1:1 to the function it tests. This keeps the "what am I testing" question answered at the class level.

4. **`monkeypatch` for env vars** — `REASONS_HF_ORG` override tests use `monkeypatch.setenv` / `monkeypatch.delenv`, the pytest-native way to mutate the environment without leaking.

5. **Fixture-scoped DB** — The `db` fixture uses `tmp_path` (pytest's per-test temp dir) and calls `api.init_db` + `api.add_node`. No shared state between tests.

6. **Call-count assertion as integration guard** — `test_upload_calls` asserts `mock_urlopen.call_count == 4`. This catches accidental extra network calls (e.g., a stray health-check) without asserting on individual payloads.

## Dependencies

**This file imports:**
- `reasons.api` — provides `init_db`, `add_node`, `publish_hf`, `import_hf`
- `reasons.hf` — provides `DEFAULT_HF_ORG`, `create_repo`, `resolve_repo_id`, `upload_file`
- `reasons.cli` — provides `cmd_pull`, `cmd_import_hf`, `cmd_publish` (smoke check only)
- `pytest`, `unittest.mock`, `urllib.error.HTTPError`, `json`, `io.BytesIO`

**Nothing else imports this file.** It is a leaf test module. The `conftest.py` in `tests/` may provide shared fixtures (e.g., `db` could be defined there too, but this file redefines a local one).

## Flow

The typical test flow (e.g., `TestCreateRepo.test_creates_repo`):

```
1. @patch replaces reasons.hf.urlopen with a MagicMock
2. mock_urlopen.return_value is set to a _mock_response()
3. The SUT is called: create_repo("test-eem", token="tok")
   Internally, reasons.hf builds a urllib.request.Request(
       url="https://huggingface.co/api/repos/create",
       data=json.dumps({name, organization, private, token}).encode(),
       method="POST"
   ) and calls urlopen(req)
4. The mock captures the Request in call_args
5. Test asserts on req.full_url, req.data (parsed JSON), etc.
```

For `TestPublishHf.test_publishes_three_files`:

```
1. db fixture creates SQLite + adds node "n1"
2. mock_urlopen.return_value = _mock_response() (empty JSON = success)
3. api.publish_hf("test-eem", token="tok", db_path=db)
   Internally:
     a. resolve_repo_id("test-eem") → "EEM-Hub/test-eem"
     b. create_repo(...) → urlopen #1
     c. Serialize network → upload_file("network.json") → urlopen #2
     d. Serialize beliefs.md → upload_file("beliefs.md") → urlopen #3
     e. Serialize README.md → upload_file("README.md") → urlopen #4
   Returns dict with repo_id, url, files_uploaded
4. Test asserts the return dict and the set of uploaded filenames
```

For error paths, `mock_urlopen.side_effect = HTTPError(...)` causes the `with` block in `reasons.hf` to raise, which the production code catches and re-raises as a `RuntimeError` with a descriptive message. The test asserts on that `RuntimeError` via `pytest.raises`.

## Invariants

- **Bare names always resolve to `EEM-Hub/<name>`.** The constant `DEFAULT_HF_ORG == "EEM-Hub"` is the single source of truth (asserted in `test_default_org_constant`).
- **Explicit `org/repo` format is never rewritten.** Neither the default org nor `REASONS_HF_ORG` overrides an explicit org prefix.
- **`REASONS_HF_ORG` env var, when set, replaces `EEM-Hub` as the default** but does not touch explicit orgs.
- **409 on repo creation is idempotent** — the function returns the repo_id as if creation succeeded. This means `publish_hf` is safe to call repeatedly.
- **401 always maps to `RuntimeError("Authentication failed…")`** regardless of which HF call (create or upload) triggers it.
- **`publish_hf` uploads exactly three files** in a deterministic set: `network.json`, `beliefs.md`, `README.md`.
- **A token is mandatory for publish.** Without one (neither parameter nor `HF_TOKEN` env var nor `~/.huggingface/token`), a `RuntimeError` is raised before any network call.

## Error Handling

All error handling in the SUT follows a consistent pattern:

| HTTP Status | SUT Behavior | Test Assertion |
|-------------|-------------|----------------|
| 401 | `RuntimeError("Authentication failed…")` | `pytest.raises(RuntimeError, match="Authentication failed")` |
| 409 (create only) | Swallowed; returns repo_id as success | `assert result == "org/repo"` |
| 4xx/5xx (other) | `RuntimeError("HTTP <code>…")` or `RuntimeError("Failed to upload…")` | `pytest.raises(RuntimeError, match="…")` |
| Non-HF URL to `resolve_repo_id` | `ValueError("Not a HuggingFace URL…")` | `pytest.raises(ValueError, match="Not a HuggingFace URL")` |

The tests never let an `HTTPError` escape to the caller — they always assert the `RuntimeError` wrapper. This confirms the SUT's contract: **callers see domain errors (`RuntimeError`), not HTTP errors.**

The `test_no_token_raises` test is notable: it patches `Path.home` to a nonexistent path to ensure the `~/.huggingface/token` fallback file isn't found, and deletes `HF_TOKEN` from the env. This is a defensive test against accidental token leakage from the dev machine.

## Topics to Explore

- [file] `reasons/hf.py` — The production module under test. Shows how `urlopen` is called, how the context-manager pattern is used, and where token resolution (param → env → file) is implemented.
- [function] `reasons/api.py:publish_hf` — The orchestration function that sequences `resolve_repo_id` → `create_repo` → three `upload_file` calls. Understanding its serialization step (network → JSON, beliefs → markdown, README) reveals what "publish" means concretely.
- [file] `tests/test_import_hf.py` — The companion test for the pull/import path. Comparing the two files clarifies what `test_hf_default_org.py` covers vs. what the import test covers.
- [file] `reasons/cli.py` — Shows how `cmd_publish` and `cmd_pull` wire argparse subcommands to `api.publish_hf` / `api.import_hf`. The `TestCLIDispatch` class here is a smoke test; the real wiring lives here.
- [general] `eem-hub-organization` — What the `EEM-Hub` HuggingFace org is, what repos live there, and the convention for naming EEM network repos (e.g., `ddia-expert`).

## Beliefs

- `resolve-repo-id-prepends-default-org` — A bare repository name (no `/`) is always prefixed with the `REASONS_HF_ORG` env var value or `EEM-Hub`, but an explicit `org/name` string is never modified.
- `create-repo-409-is-idempotent` — A 409 HTTP response from the HF "create repo" endpoint is treated as success, making `publish_hf` safe to invoke multiple times against the same repo name.
- `publish-hf-produces-exactly-three-files` — `api.publish_hf` uploads precisely `network.json`, `beliefs.md`, and `README.md` (and nothing else) to the target HF repo on the `main` branch.
- `hf-errors-are-runtime-errors` — All HuggingFace API failures (401, 500, network errors) are wrapped into `RuntimeError` with human-readable messages; raw `HTTPError` never propagates to CLI callers.
- `publish-hf-requires-token` — `api.publish_hf` raises `RuntimeError` before making any network request if no token can be resolved from the parameter, `HF_TOKEN` env var, or `~/.huggingface/token` file.
