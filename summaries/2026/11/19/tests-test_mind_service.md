---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_mind_service.py

# `tests/test_mind_service.py`

## Purpose

This file is the unit-test suite for the **agentic-mind-service synchronization** layer of the project. It exercises two symmetric operations:

- **Import** — pull a belief graph from a remote "mind service" HTTP endpoint into a local SQLite DB (`api.import_api`).
- **Export** — push local belief nodes up to the remote mind service (`api.export_api`).

Rather than exercising a live network, every test stubs out `urllib.request.urlopen` (patched at `reasons.mind_service.urlopen`) and asserts on the *HTTP request objects* and *return-value counters*. The file therefore documents the exact wire contract (URLs, headers, JSON bodies, error messages) enforced by `reasons/mind_service.py` and `reasons/api.py`.

---

## Key Components

### Module-level helpers

- **`SAMPLE_EXPORT`** — a canonical export payload with a `meta` envelope, two `nodes` (`"a"`, `"b"`), and empty `nogoods`/`repos`. The sample node `"b"` carries a `SL` justification whose antecedent is `"a"`, which lets tests re-verify serialization of justification edges. Its two-node shape drives the assertion `result["nodes_imported"] == 2`.

- **`_mock_response(data)`** — builds a `MagicMock` that emulates `urlopen`'s return value:
  - `.read()` returns `json.dumps(data).encode("utf-8")` — matching the "returns JSON string" contract of `fetch_export`.
  - Implements `__enter__`/`__exit__` (returning `False`) so `with urlopen(...) as resp:` works.
  - It does **not** mimic HTTP status codes; status handling is tested by raising `HTTPError` instead.

### Test classes (one per tested symbol)

| Class | Target | Contract verified |
|-------|--------|--------------------|
| `TestResolveConfig` | `_resolve_config(url, agent_id, api_key)` | env fallback, explicit override, trailing-slash stripping, required-field errors |
| `TestFetchExport` | `fetch_export(url, agent_id, api_key)` | URL shape, Bearer auth, returns JSON string, 401/404 mapping |
| `TestPushBelief` | `push_belief(url, agent_id, api_key, node_id, text, sl=..., source=...)` | POST URL/method, JSON body, optional field omission, Bearer auth |
| `TestImportApi` | `api.import_api(...)` | counts, `init=True` creates DB, missing-config error |
| `TestExportApi` | `api.export_api(...)` | pushes every node, forwards `sl`, per-node error counting, missing-config error |

### Per-test helper idioms

- **`mock_urlopen.call_args[0][0]`** — pulls the first positional argument (the `Request` object) from the last `urlopen` call.
- **`mock_urlopen.call_args_list`** — used in `test_export_sends_sl_justifications` to inspect *all* outbound requests and locate the derived node body.
- **`mock_urlopen.side_effect = [ok, HTTPError(...)]`** — used in `test_export_counts_errors` to fail the *second* node only, proving `export_api` catches and counts per-node errors instead of aborting.

---

## Patterns

1. **Patch-at-module-scope**: `@patch("reasons.mind_service.urlopen")` — matches the production import (`from urllib.request import urlopen`), so both `fetch_export` and `push_belief` are transparently intercepted in one place.

2. **Test-class grouping by SUT** (subject-under-test). Each class bundles the behaviors of one callable; names like `test_401_raises` are identity-only, relying on the class name to scope.

3. **Fixture-driven isolation**: `tmp_path` provides a fresh SQLite file per test; `monkeypatch.setenv` / `monkeypatch.delenv` manages `MIND_*` env vars per test without leaking.

4. **Stubbing the context-manager protocol** — `_mock_response` explicitly adds `__enter__`/`__exit__` to the `MagicMock`, a pattern worth copying whenever mocking client code that uses `with`.

5. **Side-effect sequencing** — using a *list* for `urlopen.side_effect` to inject a partial failure; this is coupled implicitly to the iteration order inside `export_api` (see Invariants).

6. **Count-based assertions** — `result["nodes_imported"]`, `result["nodes_exported"]`, `result["errors"]` are the *only* structured outputs asserted for the batch calls, indicating a "aggregate counter, not exception" contract.

---

## Dependencies

### Imports (what this file pulls in)

- **stdlib**: `json`, `io.BytesIO`, `pathlib.Path`, `unittest.mock.{MagicMock, patch}`, `urllib.error.HTTPError`
- **third-party**: `pytest` (raises, fixtures)
- **project**:
  - `reasons.api` — the public façade exposing `init_db`, `add_node`, `import_api`, `export_api`
  - `reasons.mind_service` — internal SUT for `_resolve_config`, `fetch_export`, `push_belief`
  - `MockResponse` behavior depends on `reasons.mind_service.urlopen` being a module-level name (test would fail if that import were refactored to `urllib.request.urlopen(...)` inline).

### Imported by

Nothing — this is a leaf test module. Companion tests (`conftest.py`) likely provide fixtures but this file does not appear to depend on custom fixtures (only `tmp_path`, `monkeypatch`, which are pytest built-ins).

---

## Flow

The typical execution of a single test looks like:

1. **Setup**
   - Configure mocks: `mock_urlopen.return_value = _mock_response(payload)` **or** `mock_urlopen.side_effect = [...]`.
   - For import/export tests: prepare a tmp DB path via `tmp_path / "test.db"` and call `api.init_db(db_path=...)` or pass `init=True`.

2. **Invoke SUT** — one of `_resolve_config`, `fetch_export`, `push_belief`, `api.import_api`, `api.export_api`.

3. **Inspect the captured request** (for the low-level tests):
   - `req = mock_urlopen.call_args[0][0]`
   - Assert on `req.full_url`, `req.method`, `req.get_header("Authorization")`, `json.loads(req.data)`.

4. **Assert aggregates** (for the high-level tests):
   - `result["nodes_imported"]`, `result["nodes_exported"]`, `result["errors"]`.

5. **Assert side-effects on disk** — `Path(db).exists()` after `import_api(..., init=True)`.

For `test_export_sends_sl_justifications`, the flow is a *batch inspect*: after `export_api` finishes, `mock_urlopen.call_args_list` is scanned, each request's `req.data` is JSON-decoded, the body whose `node_id == "d"` is selected, and the presence of `"a"`/`"b"` in its `sl` field is asserted. This proves justifications are serialized as a comma-separated `sl` string (matching `add_node(..., sl="a,b")`).

---

## Invariants

Concrete rules the test file encodes as guards:

- **Config resolution precedence**: explicit `url`/`agent_id`/`api_key` win over `MIND_SERVICE_URL`/`MIND_AGENT_ID`/`MIND_API_KEY` (`test_explicit_overrides_env`).
- **Required fields**: `_resolve_config` raises `RuntimeError` whose message matches `"url"` when URL is missing, and `"agent-id"` when agent id is missing (test matches are intentionally lowercase-with-hyphen).
- **URL normalization**: trailing `/` is stripped so `full_url` becomes `.../api/agents/{agent_id}/export` (no double slash).
- **Auth**: every outbound HTTP request carries `Authorization: Bearer <api_key>`.
- **Endpoint shapes**:
  - Fetch: `GET {base}/api/agents/{agent_id}/export`
  - Push:  `POST {base}/api/agents/{agent_id}/beliefs`
- **Method**: `push_belief` is always POST.
- **Body contract** for `push_belief`:
  - always includes `node_id`, `text`
  - includes `sl` only when non-empty (`test_omits_empty_sl`)
  - includes `source` when provided
- **Batch semantics of `export_api`**:
  - one outbound POST per node
  - iterate order must follow node insertion/id order so that `side_effect=[ok, HTTPError(500)]` maps to the *first* succeeding and *second* failing (`nodes_exported == 1`, `errors == 1`).
- **Batch semantics of `import_api`**:
  - the `nodes_imported` count equals the number of node dict entries in the payload (2 for `SAMPLE_EXPORT`)
  - `nogoods_imported` equals the `nogoods` list length (0 here)
  - `init=True` creates the DB file if absent.
- **Config gate**: both `api.import_api` and `api.export_api` raise `RuntimeError` matching `"Missing required config"` when env vars are cleared and no explicit params are passed.

---

## Error Handling

Three distinct layers are probed:

1. **Pre-flight validation — raise**
   `_resolve_config` raises `RuntimeError` synchronously when required configuration is absent. `pytest.raises(RuntimeError, match="url")` / `match="agent-id"` verify the *message substrings*, so these strings are part of the contract.

2. **HTTP-to-domain error mapping — raise**
   - HTTP 401 → `RuntimeError("Authentication failed")`
   - HTTP 404 → `RuntimeError("Agent not found")`
   
   Both are produced by `fetch_export` catching `HTTPError`. Tests raise `HTTPError("url", status, "...", {}, BytesIO(b""))` directly; the empty `BytesIO` supplies the (unused) error body.

3. **Per-item error swallowing — count**
   `api.export_api` **does not** propagate failures from individual `push_belief` calls. Instead it increments an `errors` counter and continues, returning `{"nodes_exported": N, "errors": M}`. `test_export_counts_errors` proves a mid-batch HTTP 500 yields `(1, 1)` rather than an exception. This is the most important design invariant for callers: batch export is *best-effort*.

4. **Config errors in the façade — raise**
   When config cannot be resolved, `import_api`/`export_api` *do* raise (`RuntimeError("Missing required config")`) instead of returning partial results — because there is nothing to iterate over.

---

## Topics to Explore

- [function] `reasons/mind_service.py:_resolve_config` — Confirm exact env-var names, trailing-slash normalization, and the wording of the two error messages; the tests certify substrings, not full text.
- [function] `reasons/mind_service.py:fetch_export` — See how `HTTPError` is mapped to domain messages and whether body bytes are surfaced anywhere (tests supply none).
- [function] `reasons/mind_service.py:push_belief` — Verify how `sl` is serialized (comma-joins, ordering) and whether `source_hash`, `date`, `metadata` are also forwarded (the tests only cover `source`).
- [function] `reasons/api.py:export_api` — Confirm iteration order over nodes matches insertion order; `test_export_counts_errors` silently depends on it.
- [file] `reasons/api.py` — Where the aggregate counters (`nodes_imported`, `nodes_exported`, `errors`) are constructed; this defines the return-shape contract the tests assert on.
- [general] `mind-service-wire-protocol` — Explore the matching endpoint on the service side (Meta schema_version `"1.0"`, `/api/agents/{id}/export`, `/beliefs`) to confirm round-trip compatibility.

## Beliefs

- `resolve-config-reads-env-vars` — `_resolve_config` reads `MIND_SERVICE_URL`, `MIND_AGENT_ID`, and `MIND_API_KEY` when explicit parameters are not supplied.
- `explicit-params-override-env` — Explicit parameters to `_resolve_config` take precedence over the corresponding `MIND_*` environment variables.
- `resolve-config-strips-trailing-slash` — `_resolve_config` normalizes the base URL by stripping a trailing `/` before returning it.
- `fetch-export-maps-http-errors` — `fetch_export` raises `RuntimeError("Authentication failed")` on HTTP 401 and `RuntimeError("Agent not found")` on HTTP 404.
- `push-belief-uses-bearer-auth-and-post` — `push_belief` sends an HTTP POST to `/api/agents/{agent_id}/beliefs` with an `Authorization: Bearer <key>` header.
- `push-belief-omits-empty-sl` — `push_belief` omits the `sl` key from the JSON body when no justification string is provided.
- `export-api-counts-errors-instead-of-raising` — `api.export_api` swallows per-node failures, returning aggregate `nodes_exported` and `errors` counters rather than propagating exceptions.
- `import-api-requires-resolvable-config` — `api.import_api` and `api.export_api` raise `RuntimeError("Missing required config")` when config resolution fails.
- `import-api-init-creates-db` — `api.import_api(..., init=True)` creates the SQLite database file when it does not yet exist.
