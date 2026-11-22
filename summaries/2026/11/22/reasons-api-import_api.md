---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: import_api in reasons/api.py

## `import_api` — Explanation

### Purpose

`import_api` is the **network-import counterpart to `export_api`**. It pulls a complete belief network from the external *agentic-mind-service* over HTTP (`GET /export`), materializes it as JSON, and installs it into the local RMS via `import_json` (which handles both SQLite and PostgreSQL backends). It exists so callers can sync a hive database by pointing at a URL / agent / key without dealing with Storage lifecycle, temp files, or backend dispatch themselves.

It closely mirrors the shape of `import_hf` (fetch → optional init → temp file → `import_json`), just with a different upstream.

### Contract

- **Preconditions**
  - `agentic-mind-service` is reachable at the resolved URL.
  - Credentials (`api_key` or config default) are valid.
  - For a **PostgreSQL** backend: either the schema already exists or `init=True` is passed.
  - For a **SQLite** backend: parent directory of `db_path` must be writable (for DB creation) and the system temp directory must be writable.

- **Postconditions**
  - Nodes, justifications, truth values, metadata, nogoods, and repos from the remote export are imported into `db_path` (or the PG database).
  - If the DB did not exist and initialization was triggered, an empty network with `meta.project_name` from the remote export is created first, then the import overwrites the metadata keys `schema_version`, `project_name`, `created_at`.
  - No leftover temp file (`finally: Path(temp_path).unlink()`).

- **Invariants / assumptions not enforced by types**
  - `_resolve_config` returns a **3-tuple** `(url, agent_id, api_key)` — signature isn't annotated.
  - `fetch_export` returns a **`str`** containing valid JSON. This function never validates JSON shape beyond the forgiving `.get("meta", {}).get("project_name", "")` access.
  - `db_path` is only meaningful for SQLite; for PostgreSQL it is a placeholder and `Path(db_path).exists()` is effectively meaningless (usually false).
  - The `init` flag semantics are **backend-dependent** (see Algorithm) — a subtle, undocumented distinction.

### Parameters

| Parameter | Meaning | Edge cases |
|---|---|---|
| `url: str \| None` | Base URL of agentic-mind-service. `None` → fallback from config/env. | Bad URLs propagate from `fetch_export`. |
| `agent_id: str \| None` | Agent identifier on the remote service. `None` → config default. | Empty/None resolution depends on `_resolve_config`. |
| `api_key: str \| None` | Auth token. `None` → config default. | Missing key typically raises inside `fetch_export` (HTTP 401/403). |
| `init: bool = False` | Whether to initialize the DB before import **if it doesn't exist**. | For SQLite, the flag is effectively ignored — a missing file is always initialized. Only meaningful for PostgreSQL. |
| `db_path: str = DEFAULT_DB` | Local SQLite path (or placeholder for PG). | Relative paths resolve against CWD. |
| `pg_conninfo=None` | PostgreSQL conninfo / truthy object. When truthy, forces PG backend dispatch in `init_db` and `import_json`. | Any truthy value triggers dispatch; actual parsing happens in the `pg` module. |
| `domain_id=None` | PG domain/tenant selector. | Passed through to `_pg_dispatch`. Ignored for SQLite. |

### Return Value

A dict. For SQLite, the shape is the one documented and produced by `import_json`:

```python
{"nodes_imported": int, "nogoods_imported": int}
```

For PostgreSQL, `import_json` returns whatever `PgApi.import_json(data=data)` returns — assumed to be the same shape, but this function does **not** normalize or validate it. Callers should not assume extra keys.

Errors are **not** returned in the dict; they propagate as exceptions.

### Algorithm

1. **Config resolution** — `_resolve_config(url, agent_id, api_key)` merges explicit args with fallback sources (env, config file) into a resolved triple.
2. **Fetch** — `fetch_export(resolved_url, resolved_agent_id, resolved_api_key)` performs `GET /export` and returns the raw JSON string.
3. **Conditional initialization** — `db_exists = Path(db_path).exists()`. Then:
   ```python
   if not db_exists and (init or not pg_conninfo):
       data = json.loads(json_str)
       project_name = data.get("meta", {}).get("project_name", "")
       init_db(db_path=db_path, force=False,
               project_name=project_name,
               pg_conninfo=pg_conninfo, domain_id=domain_id)
   ```
   - **SQLite** (`pg_conninfo` falsy): `not pg_conninfo == True`, so **an absent file triggers `init_db` regardless of `init`**. `init=True` here is a no-op for gating.
   - **PostgreSQL** (`pg_conninfo` truthy): the branch collapses to `not db_exists and init`. Because `db_path` is a placeholder, `db_exists` is normally False, so `init=True` is the only way to auto-create the PG schema.
   - If `db_exists` is True (SQLite file present), initialization is skipped entirely — `init=True` is silently disregarded.
4. **Temp file** — The JSON is written to a `NamedTemporaryFile(mode="w", suffix=".json", delete=False)`. `import_json` requires a path (it re-reads and parses from disk).
5. **Import** — `import_json(temp_path, db_path=db_path, pg_conninfo=pg_conninfo, domain_id=domain_id)` reconstructs the network: topological node insertion, exact truth-value restoration, nogoods, repos, then restores tags/sources directly via SQL.
6. **Cleanup** — `Path(temp_path).unlink()` runs unconditionally in `finally`.

**Note on double parsing**: When initialization triggers, `json_str` is parsed once for `project_name`, then written to disk and parsed **again** by `import_json`. Minor inefficiency, not a bug. In the PG-without-init path, JSON is still written to a temp file even though `import_json` immediately dispatches to PG using a parsed dict — the temp file is not strictly necessary for that branch.

### Side Effects

- **Network I/O**: `GET /export` against agentic-mind-service.
- **Filesystem**: creates a temp file under `$TMPDIR`, then deletes it in `finally`.
- **Database mutation**: creates a new SQLite/PG database if missing (per the branching above); inserts nodes / justifications / nogoods / repos / tags / sources.
- **In-memory cache**: via `_with_network(db_path, write=True)` inside `import_json` — updates `_network_cache[db_path]`.
- **PubSub**: if enabled and subscribers exist, the write path in `_with_network` publishes change events.
- **Global state**: `clear_network_cache` is *not* called; the caller may want to invalidate any stale cached `Network` reads, though `_with_network` refreshes on save.

### Error Handling

No explicit `try/except` around network or import. All failures propagate:

- `_resolve_config` → typically `RuntimeError` / `ValueError` if config is unresolvable.
- `fetch_export` → `requests`/`httpx` errors, HTTP status errors, JSON decode errors on malformed upstream responses.
- `json.loads(json_str)` (init branch) → `json.JSONDecodeError` if upstream returned non-JSON.
- `init_db` → `FileExistsError` if a concurrent process created the file between `db_exists` and `init_db` (race window).
- `tempfile.NamedTemporaryFile` → `OSError` on unwritable temp dir.
- `import_json` → `KeyError` / `ValueError` / DB errors.
- **Cleanup caveat**: if `Path(temp_path).unlink()` itself raises (e.g., Windows file lock, or the file was already removed), it will **mask** the original exception from `import_json`. In practice this includes `temp_path` being unassigned if `f.write` failed before `temp_path = f.name` — though in that case the `try` block is never entered, so it's safe.

### Usage Patterns

Typical:

```python
# SQLite: create DB if missing (init flag irrelevant), import
result = import_api(url="https://mind.example", agent_id="alice",
                    api_key="sk-...", db_path="reasons.db")

# PostgreSQL: init=True is required to auto-create schema
result = import_api(url="https://mind.example", agent_id="alice",
                    api_key="sk-...", init=True,
                    pg_conninfo="host=db dbname=rms", domain_id="d1")
```

Caller obligations:

- **PostgreSQL callers must either pass `init=True` or provision the schema themselves** — the `db_exists` file check does not detect remote state.
- Callers relying on remote `project_name` for the DB name should pass `init=True` for PG, since without init the project name is not read.
- If the caller maintains a long-lived `Network` object elsewhere, it should invalidate or reload — this function does not expose a "reload" signal beyond the pubsub events.

### Dependencies

- Stdlib: `tempfile`, `json` (module-level import in `api.py`), `pathlib.Path`.
- Internal: `reasons.mind_service._resolve_config`, `reasons.mind_service.fetch_export`.
- Internal: `reasons.api.init_db`, `reasons.api.import_json`.
- Transitive: `reasons.pg.PgApi` (via `_pg_dispatch` used inside `import_json` / `init_db`), `reasons.storage.Storage`, `reasons.pubsub`, `reasons.network.Network`.

## Topics to Explore

- [function] `reasons/api.py:import_json` — The actual network reconstruction; explains how truth values, nogoods, and tags/sources are restored on both backends.
- [function] `reasons/api.py:import_hf` — Structurally nearly identical; comparing the two reveals the shared "fetch → init → temp → import_json" pattern and where `import_hf` differs (extra `repo_id` key, `update_metadata`).
- [function] `reasons/mind_service.py:_resolve_config` — Resolves URL/agent/key from args + env/config; defines all the edge cases for the `None` fallbacks.
- [file] `reasons/mind_service.py` — Contains `fetch_export`, `push_belief`, and related auth/HTTP logic; the source of most runtime errors for `import_api`.
- [general] `pg-backend-dispatch` — How `pg_conninfo`/`domain_id` flow through `_pg_dispatch` and which API functions support PostgreSQL versus raise `NotImplementedError`.

## Beliefs

- `import-api-auto-initializes-sqlite` — For SQLite backends, `import_api` initializes a missing database unconditionally; the `init` flag does not gate that branch.
- `import-api-requires-init-for-postgres` — For PostgreSQL, `init=True` is required to auto-create the schema because the local `db_path` existence check cannot detect remote state.
- `import-api-delegates-to-import-json` — All node/nogood/repo installation is performed by `import_json`; `import_api` is only responsible for fetch, optional init, temp-file staging, and cleanup.
- `import-api-uses-temp-file-with-finally-cleanup` — The downloaded JSON is always written to a `NamedTemporaryFile(delete=False)` and removed in a `finally` block, including on import failure.
- `import-api-parses-json-twice-when-initializing` — When initialization runs, `json_str` is parsed once for `meta.project_name` and then re-parsed from the temp file by `import_json`.
