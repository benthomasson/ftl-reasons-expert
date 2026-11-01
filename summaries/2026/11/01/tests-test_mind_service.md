---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_mind_service.py

# `tests/test_mind_service.py`

## Purpose

This file validates the **agentic-mind-service sync layer** — the mechanism by which a local belief database (SQLite) exchanges belief nodes with a remote "mind service" API. It covers two directions:

- **Import**: pull a full agent export from the remote service and load it into a local DB.
- **Export**: push every node from the local DB to the remote service, one-by-one.

It also unit-tests the three low-level functions in `reasons.mind_service` (`_resolve_config`, `fetch_export`, `push_belief`) in isolation, verifying HTTP mechanics (URL construction, auth headers, JSON encoding) before the higher-level `api.import_api` / `api.export_api` integration tests exercise the full round-trip.

## Key Components

| Component | Role |
|---|---|
| `SAMPLE_EXPORT` (module-level dict) | A canonical two-node belief graph (node `a` → SL justification → node `b`) used as the mock HTTP response body. It exercises the full node schema: `text`, `truth_value`, `justifications`, `source`, `source_hash`, `date`, `metadata`. |
| `_mock_response(data)` | Builds a `MagicMock` that behaves like a `urllib` response object: `read()` returns JSON bytes, and it's usable as a context manager (`__enter__`/`__exit__`). This lets `patch("reasons.mind_service.urlopen")` return something the production code can `.read()` and `json.loads()`. |
| `TestResolveConfig` | Tests `_resolve_config`'s parameter-resolution contract (explicit > env > raise) and the trailing-slash stripping invariant. |
| `TestFetchExport` | Verifies the GET request: correct URL path (`/api/agents/{id}/export`), `Authorization: Bearer <key>` header, JSON decode of the response, and specific HTTP-error-to-`RuntimeError` mappings (401, 404). |
| `TestPushBelief` | Verifies the POST request: correct URL (`/api/agents/{id}/beliefs`), `POST` method, JSON body shape, omission of empty `sl` field, and auth header. |
| `TestImportApi` | Integration-level: calls `api.import_api` with a mocked HTTP response and a real temp SQLite DB. Asserts node counts, nogood counts, and DB file creation. |
| `TestExportApi` | Integration-level: seeds a local DB via `api.add_node`, then calls `api.export_api`. Asserts per-node POST bodies (including SL justification serialization), error counting on partial failure, and config-missing validation. |

### Contract highlights

- **`_resolve_config(url, agent_id, api_key)`** → `(str, str, str)`
  - Precedence: explicit keyword args > `MIND_SERVICE_URL` / `MIND_AGENT_ID` / `MIND_API_KEY` env vars.
  - Raises `RuntimeError` if `url` or `agent_id` is unresolvable.
  - Strips trailing `/` from the URL.

- **`fetch_export(url, agent_id, api_key)`** → `str` (JSON)
  - Issues `GET {url}/api/agents/{agent_id}/export` with `Authorization: Bearer {api_key}`.
  - Maps 401 → `RuntimeError("...Authentication failed...")`, 404 → `RuntimeError("...Agent not found...")`.

- **`push_belief(url, agent_id, api_key, node_id, text, sl=None, source=None)`** → `str` (JSON)
  - Issues `POST {url}/api/agents/{agent_id}/beliefs` with a JSON body.
  - `sl` and `source` keys are **omitted** from the body when `None`.

## Patterns

1. **HTTP mocking at the `urlopen` boundary.** Every network test patches `reasons.mind_service.urlopen` — the single I/O chokepoint in the module. This keeps tests hermetic: no socket, no real server. The `_mock_response` helper simulates the `urllib.response` interface (`.read()`, context-manager protocol) so the production code's `with urlopen(req) as resp: return resp.read().decode()` path is exercised faithfully.

2. **Layered test granularity.** The file moves from pure-unit (`TestResolveConfig` — no I/O at all) → request-shape (`TestFetchExport`, `TestPushBelief` — assert URL, method, headers, body) → integration-with-local-DB (`TestImportApi`, `TestExportApi` — real SQLite via `tmp_path`, but network still mocked). A developer can bisect a failure to the right layer.

3. **`monkeypatch` for env-var isolation.** Tests that check env-var fallback explicitly clear the relevant variables with `monkeypatch.delenv(..., raising=False)`, ensuring a clean slate regardless of the developer's shell.

4. **`tmp_path` for ephemeral databases.** The integration tests create real SQLite files under pytest's per-test temp directory, so `api.init_db`, `api.add_node`, and the import/export logic run against actual storage with no cross-test contamination.

5. **Side-effect lists for partial-failure testing.** `test_export_counts_errors` uses `mock_urlopen.side_effect = [success, HTTPError]` to simulate the first push succeeding and the second failing, verifying that `export_api` catches per-node errors and continues rather than aborting.

## Dependencies

**This file imports (directly used):**

- `reasons.api` — `import_api`, `export_api`, `init_db`, `add_node` (the public API under test)
- `reasons.mind_service` — `_resolve_config`, `fetch_export`, `push_belief` (low-level HTTP helpers)
- `urllib.error.HTTPError` — injected via `side_effect` to simulate server errors
- `unittest.mock` — `MagicMock`, `patch`
- `pytest` — `monkeypatch`, `tmp_path`, `pytest.raises`
- `json`, `io.BytesIO`, `pathlib.Path` — utilities

**What depends on this file:** Nothing at runtime. It is a leaf test module. However, it *guards* the contract of `reasons.mind_service` and `reasons.api`, so any change to those modules' signatures or error semantics will surface here first.

## Flow

### Import path (`TestImportApi`)

```
api.import_api(url, agent_id, api_key, init=True, db_path=db)
  ├─ _resolve_config(...)          → (url, aid, key)   [validated]
  ├─ fetch_export(url, aid, key)
  │    └─ urlopen(GET .../export)  → HTTP response
  │         └─ resp.read().decode()→ JSON string
  ├─ json.loads(response)          → dict with "nodes", "nogoods", "repos"
  ├─ (init=True →) init_db(db)     → creates SQLite file
  ├─ for each node in data["nodes"]:
  │    └─ insert into local DB
  └─ return {"nodes_imported": 2, "nogoods_imported": 0, ...}
```

### Export path (`TestExportApi`)

```
api.export_api(url, agent_id, api_key, db_path=db)
  ├─ _resolve_config(...)          → (url, aid, key)
  ├─ read all nodes from local DB  → list of (node_id, text, sl, source)
  ├─ for each node:
  │    ├─ push_belief(url, aid, key, node_id, text, sl=..., source=...)
  │    │    └─ urlopen(POST .../beliefs, body=JSON)
  │    ├─ success → nodes_exported += 1
  │    └─ HTTPError → errors += 1   [loop continues]
  └─ return {"nodes_exported": N, "errors": M}
```

The export is **per-node and fault-tolerant**: a 500 on node `b` does not prevent node `c` from being pushed.

## Invariants

1. **Config precedence is strict.** Explicit keyword arguments always win over environment variables. The test `test_explicit_overrides_env` locks this in.
2. **URL normalization.** A trailing slash is always stripped (`"http://localhost/"` → `"http://localhost"`), so URL concatenation with `/api/...` never produces `//api/...`.
3. **Required config is non-negotiable.** Missing `url` or `agent_id` (after both explicit and env lookup) raises `RuntimeError` — the function never proceeds with a `None` or empty string.
4. **Empty optional fields are absent, not null.** `push_belief` omits the `sl` key entirely when `sl=None`, rather than sending `"sl": null`. This matters if the remote API treats key-presence as a signal.
5. **Export is atomic-per-node, not atomic-batch.** Each node is an independent POST; partial success is a valid outcome reported via the `errors` counter.

## Error Handling

| Scenario | Behavior |
|---|---|
| `url` not resolvable (no param, no env) | `RuntimeError` with `"url"` in the message |
| `agent_id` not resolvable | `RuntimeError` with `"agent-id"` in the message |
| HTTP 401 on fetch | `RuntimeError` matching `"Authentication failed"` |
| HTTP 404 on fetch | `RuntimeError` matching `"Agent not found"` |
| HTTP 500 (or any error) on a single push during export | Caught internally; `errors` counter incremented; remaining nodes still attempted |
| HTTP 401/404 on push during export | Same per-node catch as 500 — the `test_export_counts_errors` pattern generalises |

The key architectural choice: **`fetch_export` and `push_belief` are "loud" — they raise `RuntimeError` on expected HTTP errors.** The higher-level `api.import_api` / `api.export_api` are "quiet" — they catch per-node push errors and aggregate them into a result dict. This means the caller of `export_api` never sees a partial-failure exception; it inspects `result["errors"]`.

---

## Topics to Explore

- [file] `reasons/mind_service.py` — The production module under test; read `_resolve_config`, `fetch_export`, and `push_belief` to see the exact HTTP logic these tests lock down.
- [function] `reasons/api.py:export_api` — The orchestrator that iterates over local nodes and calls `push_belief` per node; understanding its error-accumulation loop clarifies why `test_export_counts_errors` expects `nodes_exported=1, errors=1`.
- [file] `reasons/api.py` — The full public API surface (`import_api`, `export_api`, `init_db`, `add_node`); these are the "integration" entry points the tests exercise against a real SQLite file.
- [file] `tests/conftest.py` — Check for shared fixtures (e.g., a pre-seeded DB, a common `SAMPLE_EXPORT`, or autouse mocks) that might affect how these tests run in the suite.
- [general] `bearer-auth-convention` — The `Authorization: Bearer <key>` pattern is asserted in both fetch and push; trace whether `mcp_server.py` or `pubsub.py` use the same scheme to confirm a project-wide auth convention.

## Beliefs

- `push-belief-omits-empty-sl` — When `sl` is `None`, the JSON POST body to the mind-service API omits the `sl` key entirely rather than including `"sl": null`, so the remote side can distinguish "no antecedents" from "unspecified."
- `export-api-is-fault-tolerant-per-node` — `api.export_api` catches `HTTPError` on individual `push_belief` calls and increments an `errors` counter, continuing to push remaining nodes; it never aborts the batch on the first failure.
- `resolve-config-preference-explicit-over-env` — In `_resolve_config`, explicitly passed keyword arguments always take precedence over `MIND_SERVICE_URL` / `MIND_AGENT_ID` / `MIND_API_KEY` environment variables; the test suite enforces this ordering.
- `fetch-export-maps-401-to-runtimeerror` — A 401 HTTP response from the export endpoint is translated to `RuntimeError("...Authentication failed...")`, not re-raised as a raw `HTTPError`, so callers get a uniform exception type.
- `test-file-covers-both-api-layers` — This single test file validates both the low-level `reasons.mind_service` HTTP functions (request shape, headers, status mapping) and the high-level `reasons.api` orchestration (DB read/write, batch semantics), making it the primary regression guard for the entire sync path.
