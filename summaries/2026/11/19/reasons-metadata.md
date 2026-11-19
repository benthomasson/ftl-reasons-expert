---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/metadata.py

# `reasons/metadata.py`

## Purpose

This module is the single source of truth for the metadata envelope attached to exports and storage records. It defines the schema version, the generator identity string, and a builder that consistently produces a metadata `dict` for JSON export, markdown export, SQLite storage, and PostgreSQL storage.

Its responsibility is intentionally small: centralize field names, timestamp generation, and generator string formatting so exporters and persistence layers do not diverge.

## Key Components

### `SCHEMA_VERSION = "1.0"`

Module-level constant for the metadata schema version. It is hardcoded and independent of the package version returned by `_get_generator()`. Any future schema change should update this constant, but `build_meta()` itself does not validate compatibility or migrate old data.

### `_get_generator() -> str`

Private helper that returns a generator string like:

```python
"ftl-reasons/0.40.0"
```

Behavior:

- Lazy-imports `importlib.metadata.version` inside the function.
- Calls `version("ftl-reasons")`.
- On any `Exception`, falls back to `"unknown"`.
- Returns `f"ftl-reasons/{v}"`.

The lazy import avoids an import-time dependency on installed package metadata and allows the module to work from source trees or environments where the distribution is not installed.

### `build_meta(project_name="", node_count=0, created_at="") -> dict`

Factory/builder for the metadata dictionary.

Contract from the docstring:

- `project_name`: name of the belief network/project.
- `node_count`: total nodes at export/storage time.
- `created_at`: ISO 8601 timestamp of initial creation; preserved if set.

Returns a dict with exactly these six keys:

```python
{
    "schema_version": SCHEMA_VERSION,
    "project_name": project_name,
    "created_at": created_at or now,
    "updated_at": now,
    "node_count": node_count,
    "generator": _get_generator(),
}
```

Important details:

- `now` is computed once as `datetime.now(timezone.utc).isoformat(timespec="seconds")`.
- If `created_at` is missing or falsey, it receives the same `now` value as `updated_at`. For newly created metadata, `created_at == updated_at`.
- If `created_at` is truthy, it is copied verbatim without parsing or validation.
- `updated_at` is always regenerated and cannot be supplied by the caller.
- The function returns a fresh mutable dict; the module holds no global metadata state.

## Patterns

- **Single source of truth / DRY**: field names and generator formatting live only here, preventing exports and storage from drifting.
- **Builder/factory function**: `build_meta()` constructs a plain dict rather than using a dataclass or ORM model, keeping the output directly JSON-serializable and easy to persist.
- **Lazy import with defensive fallback**: `_get_generator()` imports `importlib.metadata.version` inside the function and swallows package metadata failures.
- **UTC-aware timestamp generation**: uses `datetime.now(timezone.utc)`, not naive local time.
- **Caller-preserved creation timestamp**: `created_at` is passed in from stored/exported data when available; `updated_at` is refreshed on every build.

## Dependencies

Depends only on the Python standard library:

- `datetime.datetime`
- `datetime.timezone`
- `importlib.metadata.version` inside `_get_generator()`.

It is imported by:

- `reasons/api.py`
- `reasons/cli.py`
- `reasons/export_card.py`
- `reasons/export_markdown.py`
- `reasons/pg.py`
- `reasons/storage.py`
- `reasons_lib/metadata.py`
- `tests/test_export_card.py`
- `tests/test_metadata.py`
- `tests/test_metadata_extended.py`

Because storage and export modules depend on it, changes to field names, defaults, timestamp formats, or `SCHEMA_VERSION` are cross-cutting changes. The repository also contains `reasons_lib/metadata.py`; check whether that is a re-export, mirror, or packaging shim that must stay in sync.

## Flow

1. Caller invokes `build_meta(...)` with optional project name, node count, and previously stored `created_at`.
2. `build_meta()` computes a current UTC timestamp string with second-level precision.
3. It calls `_get_generator()`.
4. `_get_generator()` attempts to resolve the installed `ftl-reasons` distribution version.
5. If resolution succeeds, it returns `ftl-reasons/<version>`.
6. If resolution fails for any `Exception`, it returns `ftl-reasons/unknown`.
7. `build_meta()` returns a new dict with the six metadata fields.

For new metadata, `created_at` and `updated_at` both use the same `now` value. For existing metadata, the caller should pass the stored `created_at` so only `updated_at` changes.

## Invariants

- `schema_version` is always exactly `SCHEMA_VERSION`, currently `"1.0"`.
- The returned dict always contains the same six keys.
- `created_at` is always a non-empty value: either the caller-provided truthy string or the generated UTC timestamp.
- `updated_at` is always the current UTC timestamp generated at call time.
- Generated timestamps are ISO 8601 strings in UTC with seconds precision, e.g. `2025-01-15T12:34:56+00:00`.
- `generator` always starts with `ftl-reasons/`.
- Caller-provided `created_at` is not normalized, parsed, or checked against `updated_at`.
- Type annotations are documentation only; `project_name`, `node_count`, and `created_at` are not runtime-validated.

## Error Handling

- The only explicit error handling is in `_get_generator()`, which catches any `Exception` from the lazy import or `version("ftl-reasons")` call and falls back to `"unknown"`.
- This prevents export/storage paths from failing just because package metadata is unavailable.
- The broad catch can hide packaging mistakes or unexpected metadata-resolution errors.
- `build_meta()` itself does not raise validation errors for bad inputs. Invalid types or malformed `created_at` values pass through and may fail later during JSON serialization, SQL binding, or downstream parsing.
- No logging is performed when version lookup fails.

## Topics to Explore

- [file] `reasons/storage.py` — Shows how the metadata dict is persisted and whether `created_at` is correctly round-tripped through SQLite/PostgreSQL.
- [file] `reasons/export_markdown.py` — Reveals which metadata fields are rendered into markdown and which are only machine-readable.
- [file] `reasons_lib/metadata.py` — Investigate whether this is a re-export, mirror, or packaging shim that must stay in sync with this module.
- [function] `reasons/metadata.py:_get_generator` — Explore package-version resolution and fallback behavior across source, editable, and installed environments.
- [general] `metadata-schema-compatibility` — Check how `schema_version = "1.0"` is consumed by importers/exporters and whether migration logic exists elsewhere.

## Beliefs

- `schema-version-is-hardcoded` — `SCHEMA_VERSION` is a module constant `"1.0"` and is not derived from the installed package version.
- `build-meta-returns-six-keys` — `build_meta` returns a dict with `schema_version`, `project_name`, `created_at`, `updated_at`, `node_count`, and `generator`.
- `build-meta-preserves-created-at` — A truthy `created_at` argument is copied verbatim; falsey values are replaced with the current UTC timestamp.
- `build-meta-refreshes-updated-at` — `updated_at` is always regenerated using `datetime.now(timezone.utc)` with seconds precision.
- `generator-version-lookup-swallows-errors` — `_get_generator` catches any `Exception` from `importlib.metadata.version("ftl-reasons")` and returns `ftl-reasons/unknown`.
