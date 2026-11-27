---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Whether a JSON schema or shared fixture defines this contract, and whether `reasons/` and `reasons_lib/` stay synchronized.

## Summary

Neither a JSON Schema nor a shared fixture defines the observed schema/metadata contract. The contract is encoded in Python and database initialization code:

- `SCHEMA_VERSION` is defined in `reasons/metadata.py:49` and written/read by storage and DB layers.
- SQLite schema initialization lives in `reasons/storage.py:_init_schema` around line 135, with `schema_version` written at `reasons/storage.py:195`.
- PostgreSQL schema checks query `information_schema.columns` at `reasons/pg.py:135-142`, and write `schema_version` at `reasons/pg.py:151`.
- Required metadata keys are checked in Python at `reasons/api.py:2127`, `reasons/import_beliefs.py:183`, and `reasons/pg.py:2250` / `2398`.

The JSON contract search returned no matches: `search_schema_contract_json` has `total_count: 0`. Also, `pyproject.toml` does not list `jsonschema` as a dependency. So there is no evidence of an external JSON Schema defining this contract.

`tests/conftest.py` is a shared fixture file, but it does not define the schema contract. It provides:

- `pytest_configure` marker registration at `tests/conftest.py:8`
- `pg_available` / `skip_no_pg` at `tests/conftest.py:13-14`
- `pg_api` fixture at `tests/conftest.py:17`

That fixture creates a `PgApi` with a unique `domain_id`, calls `api.init_db()`, and cleans up rows from `rms_*` tables. It defines PostgreSQL test lifecycle behavior, not a shared schema contract.

## `reasons/` and `reasons_lib/` synchronization

`reasons_lib` stays synchronized with `reasons` by being a backwards-compatibility shim, not by maintaining a duplicate implementation.

`reasons_lib/__init__.py:2` contains:

```python
from reasons import *  # noqa: F401,F403
```

This means top-level names exported by `reasons/__init__.py` are re-exported through `reasons_lib`. The project docs confirm this:

- `CLAUDE.md:24` says `reasons/` is importable as `import reasons` and also aliased as `reasons_lib` for backwards compatibility.
- `CLAUDE.md:35` says `reasons_lib/` is a backwards-compat shim that re-exports from `reasons/`.
- `pyproject.toml` packages both `"reasons"` and `"reasons_lib"` under `[tool.setuptools]`.

So the synchronization mechanism is import-based, not generated-file-based or duplicated-code-based. If `reasons/__init__.py` changes its exported names, `reasons_lib` follows for package-level exports.

However, there are caveats:

- `reasons/__init__.py` has no visible `__all__`, so `from reasons import *` re-exports all non-underscore module-level names, including imported names like `dataclass` and `field`.
- `from reasons import *` does not automatically create submodule aliases such as `reasons_lib.api`.
- `CHANGELOG.md:136` mentions a historical `reasons_lib.api` layer, but the observed `reasons_lib/__init__.py` does not show submodule shims.
- The observations do not include a full `reasons_lib/` tree, so submodule-level synchronization is unverified.

The `find_reasons_lib_references` search returned `0` matches, which conflicts with explicit references in `pyproject.toml` and `CLAUDE.md`; treat that search as scoped or incomplete rather than evidence that `reasons_lib` does not exist.

## Topics to Explore

- [file] `reasons/metadata.py` — Shows where `SCHEMA_VERSION` and metadata helpers are defined.
- [file] `reasons/storage.py` — Contains SQLite `_init_schema` and `schema_version` handling.
- [file] `reasons/pg.py` — Contains PostgreSQL schema checks, metadata validation, and `init_db` behavior.
- [file] `reasons_lib/__init__.py` — Demonstrates the star-import shim and its package-level re-export behavior.
- [file] `tests/conftest.py` — Confirms the shared fixture is `pg_api` lifecycle setup, not schema validation.

## Beliefs

- `json-schema-is-not-present` — `search_schema_contract_json` found 0 matches, and `pyproject.toml` has no `jsonschema` dependency.
- `schema-version-is-defined-in-python` — `SCHEMA_VERSION` comes from `reasons/metadata.py:49` and is written by `reasons/storage.py:195` and `reasons/pg.py:151`.
- `metadata-contract-requires-three-keys` — API/import/Pg code checks `schema_version`, `project_name`, and `created_at` at `reasons/api.py:2127`, `reasons/import_beliefs.py:183`, and `reasons/pg.py:2250,2398`.
- `shared-fixture-is-pg-api-lifecycle` — `tests/conftest.py:17` defines `pg_api` for unique `domain_id` setup and cleanup, not schema contract validation.
- `reasons-lib-is-star-import-shim` — `reasons_lib/__init__.py:2` uses `from reasons import *`, so top-level exports are not duplicated.
