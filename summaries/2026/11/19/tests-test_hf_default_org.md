---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_hf_default_org.py

# `tests/test_hf_default_org.py` — Explanation

## Purpose

This is a **unit test module** for the Hugging Face Hub integration in the `reasons` package. It exercises four closely related capabilities introduced together:

1. **Default-org resolution** — bare repo names like `ddia-expert` are automatically prefixed with a default org (`EEM-Hub/`) so users don't have to type the full `owner/name` on every command.
2. **`create_repo`** — idempotent creation of a Hugging Face model repo.
3. **`upload_file`** — a PUT upload of a single artefact into a repo.
4. **High-level `publish_hf` / `import_hf`** — the CLI-facing operations that create a repo and push three network artefacts, or pull a network back into a local SQLite DB.

It is not a feature module — it has no runtime consumers. It exists to lock down the **contract** of the HF layer (`reasons.hf`, `reasons.api`, and the `pull`/`publish` CLI handlers) so that downstream tooling (Users' CLI usage, `import-hf`, wiki builds, etc.) can rely on stable behaviour.

---

## Key Components

### Module-level helper

```python
def _mock_response(data=None):
    resp = MagicMock()
    resp.read.return_value = json.dumps(data or {}).encode("utf-8")
    resp.__enter__ = lambda s: s
    resp.__exit__ = MagicMock(return_value=False)
    return resp
```

Produces a **fake `urlopen` context manager** so `with urlopen(...) as resp:` works. The response body is serialized JSON (UTF-8). The `__exit__` returning `False` means exceptions raised inside the `with` block would propagate (though the tests never trigger that).

### SAMPLE_NETWORK fixture (constant)

The minimal valid `network.json` shape:
- `meta.schema_version == "1.0"`, `meta.project_name == "test-eem"`
- one node `a` with `truth_value == "IN"`
- empty `nogoods` and `repos`

This is what `import_hf` is expected to consume.

### `db` fixture (bottom of file, before `TestPublishHf`)

```python
@pytest.fixture
def db(tmp_path):
    db_path = str(tmp_path / "test.db")
    api.init_db(db_path=db_path)
    api.add_node("n1", "Test belief.", db_path=db_path)
    return db_path
```

Creates an isolated SQLite DB with one node, keyed under pytest's `tmp_path`. Used by the publish tests so `publish_hf` has something to export.

### `TestResolveRepoId` (9 tests)

Contract for `reasons.hf.resolve_repo_id`:

| input | output |
|---|---|
| `"ddia-expert"` | `"EEM-Hub/ddia-expert"` |
| `"myuser/my-eem"` | unchanged |
| `"https://huggingface.co/org/repo"` | `"org/repo"` |
| `"ddia-expert/"` | `"EEM-Hub/ddia-expert"` (trailing slash stripped) |
| `"  ddia-expert  "` | `"EEM-Hub/ddia-expert"` (whitespace stripped) |
| `"https://github.com/..."` | `ValueError("Not a HuggingFace URL")` |
| env `REASONS_HF_ORG=my-company` + `"internal-eem"` | `"my-company/internal-eem"` |
| env `REASONS_HF_ORG=my-company` + `"other/repo"` | `"other/repo"` (explicit wins) |
| `DEFAULT_HF_ORG` | `"EEM-Hub"` |

### `TestCreateRepo` (5 tests)

Contract for `reasons.hf.create_repo`:

- **POST** to `https://huggingface.co/api/repos/create`
- JSON body includes `name`, `organization` (resolved from `DEFAULT_HF_ORG` if the input is a bare name), and `private` (default `False`).
- **409 Conflict** → silently returns the repo id (idempotent).
- **401 Unauthorized** → `RuntimeError("Authentication failed")`.
- Other HTTP error (e.g. **500**) → `RuntimeError` mentioning the code (`"HTTP 500"`).
- Return value: the fully-qualified `"org/repo"` string.

### `TestUploadFile` (4 tests)

Contract for `reasons.hf.upload_file`:

- **PUT** to `https://huggingface.co/api/models/{repo_id}/upload/main/{filename}`
- `Authorization: Bearer {token}` header
- `req.data` is the raw file bytes.
- **401** → `RuntimeError("Authentication failed")`
- **500** → `RuntimeError("Failed to upload")`

### `TestPublishHf` (5 tests)

Contract for `reasons.api.publish_hf(repo, token=..., private=..., db_path=...)`:

- Returns `{"repo_id", "url", "files_uploaded"}` where `url == "https://huggingface.co/{repo_id}"`.
- `files_uploaded` is exactly `{"network.json", "beliefs.md", "README.md"}`.
- Internally issues **4** `urlopen` calls: 1 `create_repo` + 3 `upload_file`.
- `private=True` propagates through to the `create_repo` body.
- With **no token available** (env `HF_TOKEN` deleted, `~/.huggingface/token` path patched out via `Path.home`), raises `RuntimeError` matching `"token required"`.

### `TestPullCommand` (3 tests)

Contract for `reasons.api.import_hf(repo, init=True, token=..., db_path=...)`:

- Bare name `"ddia-expert"` resolves to `"EEM-Hub/ddia-expert"` and imports 1 node.
- Explicit org `"myuser/my-eem"` is preserved in the returned `repo_id`.
- The **download request URL** contains the fully-resolved repo id (`"EEM-Hub/test-eem" in req.full_url`).

### `TestCLIDispatch` (3 tests)

Pure smoke tests:
- `cmd_pull`, `cmd_publish`, `cmd_import_hf` are importable and callable.
- `reasons.cli` module exposes both `cmd_pull` and `cmd_publish`.

There is **no real assertion on the dispatch table** here — `parser = argparse.ArgumentParser()` is constructed but never used to parse a `"pull"` or `"publish"` subparser. This is a known gap.

---

## Patterns

1. **Colocated test classes** — one class per unit under test (`TestResolveRepoId`, `TestCreateRepo`, ...). Easy to grep, easy to run subset with `-k`.
2. **Mock at the network boundary** — every test patches `reasons.hf.urlopen` (never `urllib.request.urlopen` directly). This is the correct seam: `reasons.hf` imports `urlopen` into its own namespace, so patching there is stable across refactors.
3. **Side-effect-driven error simulation** — HTTP failures are injected via `mock_urlopen.side_effect = HTTPError(url, code, msg, {}, BytesIO(b""))`. This mirrors how Python's urllib actually raises on non-2xx.
4. **Context-manager-aware mock** — `_mock_response` hand-builds `__enter__`/`__exit__` instead of using `MagicMock()` bare, because `urlopen(...)` returns a `contextlib`-compatible object.
5. **Env-var and filesystem isolation** — `monkeypatch` deletes `HF_TOKEN` and `monkeypatch.setenv` sets `REASONS_HF_ORG`; a `Path.home` patch neutralizes the fallback token file so the "no token" test does not depend on the developer's laptop state.
6. **Argument introspection** — assertions inspect the request object via `mock_urlopen.call_args[0][0]` (single call) and `mock_urlopen.call_args_list[0][0][0]` (first of several). The body is checked by `json.loads(req.data)`.
7. **Idempotency assertion** — `test_already_exists_ignored` encodes an important behaviour: 409 is not an error.

---

## Dependencies

### Imports (what this file pulls in)
- `json`, `BytesIO`, `unittest.mock` — test plumbing.
- `urllib.error.HTTPError` — constructing the error fixtures.
- `pytest` — test runner, fixtures, `pytest.raises`.
- `reasons.api` — the high-level `publish_hf`, `import_hf`, `init_db`, `add_node`.
- `reasons.hf` — the low-level `DEFAULT_HF_ORG`, `create_repo`, `resolve_repo_id`, `upload_file`.
- `reasons.cli` — the CLI handlers `cmd_pull`, `cmd_import_hf`, `cmd_publish`, `main`.

### Consumers of this file
This file has **no imports from other source code** — it is imported by pytest only. It is the sole authoritative spec for the default-org behaviour, so upstream tools that rely on it include:
- `reasons/api.py` (`publish_hf`, `import_hf`) — behaviour under test.
- `reasons/hf.py` (`resolve_repo_id`, `create_repo`, `upload_file`) — behaviour under test.
- `reasons/cli.py` (`cmd_pull`, `cmd_publish`) — smoke-tested.

Because `reasons.hf.urlopen` is patched, this test also implicitly pins the module's *import name* for `urlopen` — you cannot replace `urllib.request.urlopen` with a custom wrapper without updating the patch target.

---

## Flow

### Data flow in `TestPublishHf`
```
pytest db fixture (SQLite on disk)
        │
        ▼
api.publish_hf("test-eem", token="tok", db_path=db)
        │
        ├── resolve_repo_id("test-eem") -> "EEM-Hub/test-eem"
        ├── create_repo(repo_id, token) ──► POST /api/repos/create
        │        (urlopen mocked, returns _mock_response({}))
        ├── export network.json / beliefs.md / README.md from DB
        ├── upload_file(...) x3 ──► PUT /api/models/EEM-Hub/test-eem/upload/main/{file}
        └── returns {"repo_id", "url", "files_uploaded"}
```

### Data flow in `TestPullCommand`
```
urlopen mocked to return SAMPLE_NETWORK for
    GET https://huggingface.co/EEM-Hub/ddia-expert/resolve/main/network.json
        │
        ▼
api.import_hf("ddia-expert", init=True, token="tok", db_path=db)
        │
        ├── resolve_repo_id("ddia-expert") -> "EEM-Hub/ddia-expert"
        ├── download network.json
        ├── init_db(db_path) + insert nodes
        └── returns {"repo_id": "EEM-Hub/ddia-expert", "nodes_imported": 1}
```

### Control flow in the test runner
Each test method:
1. Applies decorators (`@patch("reasons.hf.urlopen")`).
2. Builds the input (fixture, `_mock_response`, or raw args).
3. Calls the SUT.
4. Asserts on either return value (`result["repo_id"]`, etc.) or captured `urlopen` request (`req.full_url`, `req.get_method()`, `req.data`, `req.get_header("Authorization")`).

---

## Invariants

These are *enforced by tests*, so changing the implementation must not violate them:

- `DEFAULT_HF_ORG == "EEM-Hub"` — a literal constant.
- **Bare name → default org prefix.**
- **Explicit `owner/repo` never gets prefixed or overridden by `REASONS_HF_ORG`.**
- **`REASONS_HF_ORG` overrides `DEFAULT_HF_ORG` but only for bare names.**
- **Whitespace and trailing slashes are stripped before parsing.**
- **Only `https://huggingface.co/...` URLs are accepted by `resolve_repo_id`** — everything else raises `ValueError`.
- **`create_repo` is idempotent on 409.**
- **`upload_file` always issues a PUT with `Authorization: Bearer <token>`.**
- **`publish_hf` always uploads exactly three files** — `network.json`, `beliefs.md`, `README.md`.
- **Token resolution order includes `HF_TOKEN` and `~/.huggingface/token`** — both must be unavailable to trigger `"token required"`.
- **`import_hf` output dict always contains `repo_id` (fully resolved) and `nodes_imported`.**

Note: the file imports `call` but never uses it, and imports `argparse.ArgumentParser` in `test_commands_registered` without asserting on the built parser — the CLI dispatch assertions are weaker than the HF assertions.

---

## Error Handling

| Failure | Where | Behaviour |
|---|---|---|
| Non-HF URL passed to `resolve_repo_id` | `TestResolveRepoId.test_non_hf_url_raises` | `ValueError("Not a HuggingFace URL")` |
| Repo already exists (409) on `create_repo` | `test_already_exists_ignored` | swallowed, returns repo id |
| 401 from `create_repo` | `test_auth_failure_raises` | `RuntimeError("Authentication failed")` |
| 5xx from `create_repo` | `test_other_error_raises` | `RuntimeError` containing `"HTTP 500"` |
| 401 from `upload_file` | `test_auth_failure_raises` | `RuntimeError("Authentication failed")` |
| 5xx from `upload_file` | `test_upload_failure_raises` | `RuntimeError("Failed to upload")` |
| No token in env or file | `test_no_token_raises` | `RuntimeError("token required")` |

Convention: **HTTP status → specific exception message**; only 409 is swallowed. Tests use `pytest.raises(..., match=...)` so the message text is part of the contract — refactoring the wording will break the suite.

---

## Topics to Explore

- [file] `reasons/hf.py` — The module under test: `resolve_repo_id`, `create_repo`, `upload_file`, `DEFAULT_HF_ORG`; the token-resolution ladder (`HF_TOKEN` env → `~/.huggingface/token`).
- [function] `reasons/api.py:publish_hf` — Verifies the 1-create + 3-upload contract; check how the three artefacts (network.json, beliefs.md, README.md) are assembled before upload.
- [function] `reasons/api.py:import_hf` — Pulls `network.json` from Hub, runs `init_db` and inserts nodes; understand the `init=True` flag semantics and `nodes_imported` accounting.
- [file] `reasons/cli.py` — The smoke-tested CLI dispatch; verify that `cmd_pull` really delegates to `cmd_import_hf` and that `cmd_publish` mirrors `cmd_pull`'s wiring.
- [general] `hf-token-resolution` — Trace how `publish_hf` and `import_hf` obtain the token (env var vs. credential file) and whether `resolve_repo_id` is called in both paths.
- [file] `tests/test_import_hf.py` — Complements this file by covering the import-hf path more deeply (file layout, 404 handling, model card parsing).

---

## Beliefs

- `default-org-is-eem-hub` — `DEFAULT_HF_ORG` is the string `"EEM-Hub"` and bare repo names are prefixed with it by `resolve_repo_id`.
- `env-org-override-does-not-apply-to-explicit-orgs` — Setting `REASONS_HF_ORG` changes the default org for bare names only; explicit `owner/repo` inputs are returned unchanged.
- `resolve-repo-id-rejects-non-hf-urls` — `resolve_repo_id` validates that URL inputs start with `https://huggingface.co/` and raises `ValueError("Not a HuggingFace URL")` otherwise.
- `create-repo-is-idempotent-on-409` — A `HTTPError(..., 409, ...)` from the create endpoint is swallowed and the repo id is returned as if creation succeeded.
- `publish-hf-produces-three-files` — `publish_hf` uploads exactly `{"network.json", "beliefs.md", "README.md"}` and issues 4 `urlopen` calls (1 create + 3 uploads).
- `upload-file-uses-put-and-bearer-auth` — `upload_file` sends a PUT to `/api/models/{repo_id}/upload/main/{filename}` with header `Authorization: Bearer <token>`.
- `publish-hf-requires-token` — `publish_hf` raises `RuntimeError("token required")` when neither `HF_TOKEN` nor `~/.huggingface/token` supplies a token.
