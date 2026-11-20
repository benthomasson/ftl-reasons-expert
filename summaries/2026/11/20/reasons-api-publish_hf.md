---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: publish_hf in reasons/api.py

## Explanation of `publish_hf`

### Purpose

`publish_hf` is the top-level "export and ship" operation for the Reason Maintenance System. It packages a local (or Postgres-backed) belief network into three artifacts — a canonical JSON export, human-readable `beliefs.md`, and a HuggingFace "EEM" card (`README.md`) — and pushes them to a HuggingFace Hub repository so the network can be consumed, versioned, and re-imported (via `import_hf`) by others.

It exists so every publishing path (CLI, scripts, LangGraph tools) has one function that resolves auth, creates the destination repo, and uploads a consistent snapshot in one call.

### Contract

**Preconditions**
- A HuggingFace token must be resolvable. The function itself never fabricates one — it delegates to `.hf._resolve_token`, which checks: the `token` argument, then the `HF_TOKEN` env var, then the CLI cache (`huggingface-cli login`).
- `db_path` must point to a readable DB (or `pg_conninfo` must be provided and reachable).
- Network access to `huggingface.co` must be available.

**Postconditions (on success)**
- A HuggingFace repo exists at `parsed_id` (created if missing; existing repos are reused).
- Exactly three files exist in the repo: `network.json`, `beliefs.md`, `README.md`.
- The returned dict contains the resolved repo ID, the public URL, and the list of files uploaded.

**Invariants**
- The three artifacts are produced from the same set of beliefs (the same `visible_to` filter is applied to each). For SQLite, this is cheap because exports share the module-level `_network_cache`. For Postgres, the three exports are separate round-trips and are *not* guaranteed to see the same snapshot — this is an implicit assumption, not enforced by code.
- No rollback: once the first file is uploaded, a later failure leaves a partially-populated repo.

### Parameters

| Parameter | Type | Meaning | Edge cases |
|---|---|---|---|
| `repo_id` | `str` | HF repo identifier. Accepts a bare name (resolved against a default org), `user/repo`, or a URL. | Bare-name resolution is delegated to `.hf.resolve_repo_id` (see `tests/test_hf_default_org.py`). |
| `token` | `str \| None` | Explicit HF auth token. | `None` → falls back to env / cache. If no fallback, raises. |
| `private` | `bool` | Create the repo as private. | Only honored when the repo is newly created; existing repos are not retargeted. |
| `visible_to` | `list[str] \| None` | Access-tag filter — only nodes whose `access_tags` are a subset are exported. | `None` = export everything. Filter is applied per-node at export time. |
| `domain` | `list[str] \| None` | Passed straight into `export_card` and rendered into the README. | Type is annotated `list[str]`, but the card renderer treats it loosely; inspect `export_card` to confirm shape. |
| `license` | `str` (default `"mit"`) | License string rendered into the README card. | Does **not** set the actual HF repo license metadata — card text only. |
| `base_network` | `str \| None` | Upstream network the repo was derived from (for provenance in card). | Informational only. |
| `source_repos` | `list[str] \| None` | Source repositories cited in the card. | Informational only. |
| `db_path` | `str` | SQLite path; default `reasons.db`. | Must exist unless `pg_conninfo` is set. |
| `pg_conninfo` | any | Postgres connection; when truthy, exports go through `_pg_dispatch` in the export helpers. | Mutually exclusive with local SQLite semantics. |
| `domain_id` | any | Postgres tenant/domain selector, forwarded only. | Only meaningful when `pg_conninfo` is set. |

### Return Value

```python
{
    "repo_id": parsed_id,      # str — fully-resolved HF repo id
    "url": f"https://huggingface.co/{parsed_id}",
    "files_uploaded": ["network.json", "beliefs.md", "README.md"],
}
```

If any step raises, no dict is returned. Callers should treat a raised exception as "publish did not complete," not "publish may have partially succeeded."

### Algorithm

1. **Lazy import** the HuggingFace helpers (`_resolve_token`, `create_repo`, `resolve_repo_id`, `upload_file`) from `.hf`. This keeps the module importable without any HF SDK installed as long as the user never publishes.
2. **Resolve auth**: call `_resolve_token(token)`. If the result is falsy, raise `RuntimeError` with a hint pointing at `huggingface-cli login`, `HF_TOKEN`, or `--token`.
3. **Resolve the repo ID**: `resolve_repo_id(repo_id)` — turns a bare name into `org/name` and normalizes URLs to ids.
4. **Create the repo**: `create_repo(parsed_id, token=..., private=private)`. Assumed to be idempotent — succeeds even if the repo already exists.
5. **Build the backend kwargs** dict once: `{db_path, pg_conninfo, domain_id}` — then splat it into every export helper.
6. **Export JSON**: `export_network(visible_to=..., **backend)` → `json.dumps(..., indent=2, sort_keys=True)`. `sort_keys=True` gives deterministic output for diffing/hashing across publishes.
7. **Export markdown**: `export_markdown(visible_to=..., **backend)`.
8. **Export card**: `export_card(visible_to=..., domain=..., license=..., base_network=..., source_repos=..., **backend)`.
9. **Upload sequentially** in a fixed order (`network.json`, `beliefs.md`, `README.md`), encoding each as UTF-8 bytes and appending to `files` only after `upload_file` returns.
10. **Return** the summary dict.

### Side Effects

- **Network I/O**: three sequential `PUT` calls to HF Hub (repo create + one per file).
- **Remote state mutation**: creates the repo if absent; overwrites any existing files of the same name.
- **Local reads**: SQLite reads via `_with_network` (shared, cached), or `_pg_dispatch` for Postgres.
- **Token resolution side effects**: `_resolve_token` reads env and the HF CLI cache; it does not persist anything.
- **No local filesystem writes** — exports are strings in memory, not files on disk.

### Error Handling

- **`RuntimeError`** — raised up front if no token can be resolved. This is the only *intentional* exception in this function.
- **Exceptions from `resolve_repo_id` / `create_repo` / `upload_file`** — propagated as-is. Typically HF SDK exceptions (`HfHubHTTPError`, `RepositoryNotFoundError`, auth failures). Not wrapped, not downgraded, not logged.
- **Exceptions from export helpers** (`export_network`, `export_markdown`, `export_card`) — propagated. If these fail *after* `create_repo` succeeded but *before* any upload, the repo exists but is empty.
- **Partial uploads** — if `upload_file("network.json", …)` succeeds and a later upload fails, the exception propagates and the remote repo is left in an inconsistent state. There is **no try/except, no cleanup, no retry**. This is the biggest production concern: the function is not atomic.

### Usage Patterns

Typical call from the CLI or a script:

```python
result = publish_hf(
    repo_id="reason-networks/logic-v1",   # or just "logic-v1" for the default org
    private=True,
    visible_to=["public"],
    domain=["logic", "tms"],
    license="apache-2.0",
    source_repos=["github.com/me/logic-notes"],
    db_path="reasons.db",
)
print(result["url"])
```

**Caller obligations**
- Handle exceptions — anything raised means "publish failed" and the caller may need to re-run (after the repo is now created, `private=True` on a retry becomes a no-op).
- Ensure the working DB contains only nodes visible to the caller (the `visible_to` filter does the access check for you).
- If publishing to Postgres, be aware the three exports are independent reads — snapshot consistency is not guaranteed.

### Dependencies

- `.hf` — `_resolve_token`, `create_repo`, `resolve_repo_id`, `upload_file` (lazy-imported).
- `publish_hf` itself calls:
  - `export_network` (defined in the same file) — respects `visible_to`, `pg_conninfo`, `domain_id`.
  - `export_markdown` (same file) — builds the `beliefs.md` string.
  - `export_card` (same file) — builds the `README.md` EEM card.
- `json` (stdlib), `DEFAULT_DB` (module constant).

### Assumptions Not Enforced by Types

- `create_repo` is assumed idempotent — the function never checks whether repo creation succeeded before uploading.
- `private` is assumed to be safe on retry against existing repos (in practice it is ignored, not enforced).
- `license` only affects the README card text — repo-level license metadata is not set anywhere in this function.
- The three artifacts are assumed to reflect the same belief snapshot; this is true locally due to `_network_cache`, but false across three separate Postgres queries.
- `domain: list[str] | None` is passed opaquely to `export_card`; its actual accepted shape is defined there, not here.
- No deduplication of `source_repos` / `visible_to`; callers must pass already-normalized values.

## Topics to Explore

- [function] `reasons/hf.py:_resolve_token` — Defines token precedence (argument → `HF_TOKEN` → CLI cache); the exact order matters for both `publish_hf` and `import_hf`.
- [function] `reasons/hf.py:create_repo` — Confirms whether creation is truly idempotent and how `private` behaves on an existing repo.
- [function] `reasons/hf.py:upload_file` — Determines retry/error behavior; a mid-sequence failure here leaves partial remote state.
- [file] `reasons/export_card.py` — How `domain`, `license`, `base_network`, and `source_repos` are rendered into the EEM card.
- [general] `hf-default-org` — Corresponds to `tests/test_hf_default_org.py`; explains how a bare `repo_id` (no slash) is resolved to a full `org/name`.

## Beliefs

- `publish-hf-requires-token` — `publish_hf` raises `RuntimeError` before any network or export I/O when `_resolve_token` returns falsy.
- `publish-hf-produces-three-artifacts` — On success `publish_hf` uploads exactly `["network.json", "beliefs.md", "README.md"]` in that fixed order and reports them in `files_uploaded`.
- `publish-hf-does-not-rollback` — `publish_hf` has no try/except or cleanup; a mid-sequence upload failure leaves the remote repo partially populated.
- `publish-hf-private-is-create-only` — The `private` argument is forwarded only to `create_repo` and has no effect on already-existing repos.
- `publish-hf-delegates-hf-io-to-hf-module` — All HuggingFace interaction (token resolution, repo id normalization, creation, upload) is delegated to `reasons/hf.py`; `publish_hf` itself contains no HF-specific API code.
