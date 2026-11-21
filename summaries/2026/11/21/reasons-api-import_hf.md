---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: import_hf in reasons/api.py

## Explanation of `reasons.api.import_hf`

### 1. Purpose
`import_hf` is a one-call convenience for "pull a belief network published on HuggingFace into a local (or PostgreSQL) Reason Maintenance System database." It glues three existing operations together:

1. **Fetch** the network JSON from a HuggingFace repo (via `reasons.hf.download_network`).
2. **Optionally create** the target database (via `init_db`) if it doesn't exist yet.
3. **Import** the JSON into the network (via `import_json`), using the temp-file + parse path that `import_json` currently requires.

It exists so the CLI (`reasons import-hf …`), LangGraph tools, and scripts can share a stable, single entry point that handles onboarding ("empty machine, just clone the network") without the caller having to know about DuckDB/SQLite lifecycle, JSON staging, or HuggingFace auth.

### 2. Contract

**Preconditions**
- `repo_id` must be a valid HuggingFace repo identifier (bare `user/repo` or a URL form accepted by `download_network` / `_parse_repo_id`).
- If the repo is private, `token` must be a valid HF token (or the caller must have a cached token that HF libraries pick up).
- The downloaded content must be a JSON document produced by `export_network` — `import_json` requires `nodes`, optional `nogoods`, optional `repos`, optional `meta`.
- `db_path` must be a writable location (for SQLite). For PostgreSQL, `pg_conninfo` must be a usable connection string and `domain_id` should identify the target domain.

**Postconditions**
- On success, the new nodes (and nogoods, tags, sources, repos) have been written to the target storage via `import_json`.
- The return dict reports counts plus the resolved `repo_id`.
- The staged temporary file is deleted.

**Invariants**
- The temp file is always removed, even if `import_json` raises (guaranteed by `try/finally`).
- The function does not double-import existing node IDs — `import_json` skips IDs already present.

### 3. Parameters

| Parameter | Type | Notes / Edge cases |
|---|---|---|
| `repo_id` | `str` | Passed verbatim to `download_network`. Result is later normalized through `_parse_repo_id`, so a URL form like `https://huggingface.co/u/r` will come back as `u/r` in the result dict. |
| `init` | `bool` (default `False`) | Only meaningful for the PostgreSQL backend. For SQLite backend it is effectively ignored — see the surprising logic below. |
| `token` | `str \| None` | Forwarded to `download_network` only. Not used for the import. `None` relies on whichever default token resolution `download_network` implements. |
| `db_path` | `str` (default `reasons.db`) | Path used both for `Path(db_path).exists()` and (if SQLite) for the actual import. This existence check runs even when using PostgreSQL — a stale local file named `reasons.db` will make it "think" the DB exists. |
| `pg_conninfo` | anything truthy/falsy | When truthy, all DB work is dispatched to `reasons.pg.PgApi` (`init_db` and `import_json` both route through `_pg_dispatch`). Local `db_path` is still touched for existence but not for storage. |
| `domain_id` | (opaque) | Only used when `pg_conninfo` is set — forwarded to `_pg_dispatch`. |

**Surprising behavior worth flagging:** the test

```python
if not db_exists and (init or not pg_conninfo):
```

means that for the SQLite backend (`pg_conninfo` falsy), `init` is irrelevant: **the DB is always auto-initialized when `reasons.db` is missing**, even with `init=False`. The `init` flag only matters when using PostgreSQL. The docstring ("Initialize reasons.db before import if it doesn't exist") is therefore slightly misleading for the SQLite case.

### 4. Return Value

On success, returns the dict produced by `import_json` with one extra key:

```python
{
    "nodes_imported": int,      # from import_json
    "nogoods_imported": int,    # from import_json
    "repo_id": str,             # normalized by reasons.hf._parse_repo_id
}
```

The caller must handle:
- Empty/zero counts (repo returned a valid but empty network).
- `repo_id` may differ from the input string (e.g. URL → `user/repo`, or `user` → `default-org/user` if `_parse_repo_id` applies a default org — see `tests/test_hf_default_org.py`).

### 5. Algorithm

1. **Lazy imports** of `json` (as `json_mod`) and `tempfile`, and lazy `from .hf import download_network`. This keeps the heavy HuggingFace dependency out of module import time (important for CLI startup).
2. **Download** the JSON text: `json_str = download_network(repo_id, token=token)`. This is the only network call in the function; failures propagate.
3. **Existence check**: `Path(db_path).exists()`. Note this is a *local file* check even under PostgreSQL.
4. **Auto-init branch** — if the local file is missing AND (`init` is True OR we're on SQLite):
   - `data = json.loads(json_str)` — parse once just to read `meta.project_name` (via `.get` chains, so missing meta is safe).
   - `init_db(db_path=…, force=False, project_name=…, pg_conninfo=…, domain_id=…)`. `force=False` means a race that *creates* the file between the existence check and this call raises `FileExistsError`. On PG, `init_db` dispatches to `_pg_dispatch(..., "init_db", project_name=project_name)`.
5. **Stage** the JSON into a `NamedTemporaryFile(mode="w", suffix=".json", delete=False)`. `delete=False` is deliberate so `import_json` can open it by path on any platform.
6. **Import** via `import_json(temp_path, db_path=…, pg_conninfo=…, domain_id=…)`. `import_json` opens the file, parses, adds nodes in topological order, restores truth values and timestamps, imports nogoods and repos, and restores tags/sources from the JSON.
7. **Cleanup** in `finally`: `Path(temp_path).unlink()`. If unlink itself fails (e.g. permission), it will override any in-flight exception from step 6 — worth knowing when debugging.
8. **Normalize** the returned `repo_id`: `result["repo_id"] = _parse_repo_id(repo_id)` (a private helper from `reasons.hf`) and return.

### 6. Side Effects

- **Network I/O**: one HTTPS call to HuggingFace for the file contents (via `download_network`); may also fetch repo metadata / do auth token resolution depending on the HF client.
- **Local filesystem**: creates a `NamedTemporaryFile` in the OS temp dir and deletes it.
- **File creation** (`init_db`): when the DB path didn't exist, this creates a new SQLite file (or a new PG schema). Note: `init_db` also creates a fresh `Storage` for the new SQLite file, which closes right away — this is what sets up the schema the import will write to.
- **Database mutation**: `import_json` opens a write-mode `_with_network(db_path, write=True)` context manager, mutates the in-memory `Network`, then saves and publishes change events for the network cache.
- **Pubsub events**: `_with_network(write=True)` will publish truth-value change events (via `pubsub.publish`) if there are subscribers for the DB path. Importers who care will fire cascade-change notifications even on first import.
- **Network cache update**: `_with_network` updates `_network_cache[db_path]` with the new mtime after the write.
- **Errors from cleanup**: `Path(temp_path).unlink()` in the `finally` block could raise an `OSError` that would replace the original exception — no swallowing there.

### 7. Error Handling

This function does **not** catch or translate exceptions. Anything raised by its callees propagates:

- `download_network` — HF auth failures, network errors, 404s, revoked tokens, malformed IDs.
- `json_mod.loads(json_str)` inside the auto-init branch — `json.JSONDecodeError` if the file on HF is not valid JSON. Note: this only runs *if* DB initialization is needed. If a DB already exists and the file is invalid, the failure happens later inside `import_json` instead — so the same invalid file produces different stack traces depending on state.
- `init_db` — `FileExistsError` if a file appeared between the existence check and the call (race); `_pg_dispatch` may raise connection errors if `pg_conninfo` is bad.
- `import_json` — raises on a missing temp file (shouldn't happen), `JSONDecodeError`, or anything the internal `_with_network` can raise.
- `_parse_repo_id` — can raise for a totally malformed `repo_id`, *after* the import has already succeeded. So a bad-but-downloadable id could import successfully and still throw on the last line.

The only guarantee is **cleanup**: the temp file is unlinked in `finally`. No retries, no partial-rollback, no wrapping of exceptions into a returned dict.

### 8. Usage Patterns

Typical CLI-style calls:

```python
# Onboarding a brand new local DB from a public repo
import_hf("acme/network", init=True, db_path="reasons.db")

# Against a private HF repo
import_hf("acme/private-network", token=os.environ["HF_TOKEN"], init=True)

# Against PostgreSQL (init=True required to auto-create the schema)
import_hf("acme/network", init=True, pg_conninfo="postgres://…", domain_id="acme")

# Re-sync / merge an updated export from an existing DB — safe, existing IDs skipped
import_hf("acme/network", db_path="reasons.db")  # init irrelevant here
```

Caller obligations:
- Verify the local `db_path` you passed is what you think it is — especially on the PG path, where `reasons.db` existence is checked but never used for storage.
- Be aware that on SQLite, missing DBs are **auto-created**; a typo in `db_path` will silently produce a new DB rather than the "file not found" you might expect.
- If you rely on `repo_id` being the raw string, normalize expectations via `_parse_repo_id` yourself (it does default-org resolution in `tests/test_hf_default_org.py`).

### 9. Dependencies

- **Internal** (same package):
  - `reasons.hf.download_network` — the only download path.
  - `reasons.hf._parse_repo_id` — private helper (assumed to remain callable; not part of a public `__all__`).
  - `reasons.api.import_json` — actual import implementation.
  - `reasons.api.init_db` — DB initialization, itself bridging SQLite (`Storage`) and PG (`_pg_dispatch` / `PgApi`).
  - Indirectly: `reasons.storage.Storage`, `reasons.pg.PgApi`, `reasons.pubsub`, `reasons.network.Network`.
- **Stdlib**: `json`, `tempfile`, `pathlib.Path`.
- **External**: whatever `reasons.hf` uses to talk to the HuggingFace Hub.

### Assumptions not enforced by the type system

- `download_network` returns a *string* of valid JSON (not bytes, not a dict). The signature doesn't say this, and a future change to return dicts would silently break the temp-file write and re-parse.
- The downloaded JSON contains a `meta.project_name` string — code uses `.get("meta", {}).get("project_name", "")` so it degrades gracefully, but the value is then passed into `init_db` as `project_name` and becomes the network's project label.
- The temp file path is accessible by `import_json` after the `with` block closes — guaranteed only because `delete=False` is set. `NamedTemporaryFile` with `delete=True` would break this.
- `pg_conninfo` truthiness is a reliable proxy for "PG backend is in use." The condition `not db_exists and (init or not pg_conninfo)` mixes a *local file* existence check with a *backend-selection* flag — the two are semantically orthogonal.
- `_parse_repo_id` is idempotent and handles both bare `user/repo` and URL forms, with default-org fallback behavior (see `tests/test_hf_default_org.py`).

## Topics to Explore

- [file] `reasons/hf.py` — Home of `download_network`, `_parse_repo_id`, `resolve_repo_id`, and token resolution; explains default-org behavior and private-repo auth.
- [function] `reasons/api.py:import_json` — The real import engine (topological node ordering, truth-value restoration, tag/source restoration). All `import_hf` behavior is inherited from this.
- [function] `reasons/api.py:init_db` — Schema creation and PG-vs-SQLite dispatch; needed to understand why PG requires `init=True` while SQLite auto-creates.
- [file] `tests/test_import_hf.py` — Behavioral tests for this function; useful for spotting edge cases around auto-init and temp-file cleanup.
- [general] `huggingface-hub-auth` — How `token=None` falls back to `HF_TOKEN` env or cached credentials; relevant when the caller omits `token`.

## Beliefs

- `import-hf-auto-initializes-sqlite-regardless-of-init` — For the SQLite backend (`pg_conninfo` falsy), `import_hf` always calls `init_db` when `db_path` is missing, even if `init=False`.
- `import-hf-requires-init-for-postgres` — When `pg_conninfo` is set, `import_hf` only initializes the schema if `init=True`, because the condition is `init or not pg_conninfo`.
- `import-hf-stages-via-named-temporary-file` — The downloaded JSON is written to a `NamedTemporaryFile(delete=False)` and cleaned up in a `finally` block, so `import_json` can consume it by path.
- `import-hf-delegates-download-to-hf-download-network` — All HuggingFace I/O is performed by `reasons.hf.download_network`; `import_hf` itself makes no direct HTTP calls.
- `import-hf-returns-normalized-repo-id` — The `repo_id` in the returned dict is the output of `reasons.hf._parse_repo_id`, not the raw `repo_id` argument.
