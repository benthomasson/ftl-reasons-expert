---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: build_meta in reasons/metadata.py

# `build_meta`

**Location:** `reasons/metadata.py`  
**Full signature:** `def build_meta(project_name: str = "", node_count: int = 0, created_at: str = "") -> dict:`

## Purpose

`build_meta` is the single helper for constructing the metadata dictionary used by exports and storage. It exists so JSON export, Markdown export, and SQLite/PostgreSQL persistence all use the same field names, schema version, timestamp rules, and generator string.

It records:

- schema version
- project name
- original creation timestamp
- current update timestamp
- node count
- generator/package version string

## Contract

### Preconditions

There are no enforced runtime preconditions. Callers are expected, but not required, to provide:

- `project_name`: a string, possibly empty
- `node_count`: an integer, normally non-negative
- `created_at`: either an empty string or a valid ISO 8601 timestamp string

The function does not validate any of these expectations.

### Postconditions

`build_meta` returns a fresh `dict` with exactly these keys:

- `schema_version`
- `project_name`
- `created_at`
- `updated_at`
- `node_count`
- `generator`

The values are:

- `schema_version` is always `SCHEMA_VERSION`, currently `"1.0"`.
- `project_name` is the input value.
- `created_at` is the input value if it is truthy; otherwise it is the current UTC timestamp.
- `updated_at` is always the current UTC timestamp computed during this call.
- `node_count` is the input value.
- `generator` is produced by `_get_generator()`.

### Invariants

- The returned dictionary always uses the same six field names.
- The function does not mutate its arguments.
- The function does not mutate module-level state.
- A new dictionary is returned on every call.
- If `created_at` is omitted or empty, then `created_at == updated_at` for that call.
- The function does **not** guarantee that `created_at` is valid ISO 8601.
- The function does **not** guarantee that `created_at <= updated_at`.
- The function does **not** guarantee that `node_count` is non-negative.
- The function does **not** guarantee that `project_name` is non-empty.

## Parameters

### `project_name: str = ""`

The name of the belief network/project.

Expected type is `str`. The default is an empty string. No trimming, validation, or normalization is performed. An empty string is accepted and stored as-is.

### `node_count: int = 0`

Total number of nodes at the time of export.

Expected type is `int`. No validation prevents negative values. Python’s `bool` is a subclass of `int`, so `True` or `False` would technically be stored as `1` or `0` if passed.

### `created_at: str = ""`

ISO 8601 timestamp representing when the metadata was initially created.

The code uses `created_at or now`, not `created_at is None`. Therefore:

- `""` is replaced with `now`.
- `None` is also replaced with `now`, despite the type hint saying `str`.
- Any truthy value is preserved unchanged.
- A malformed but truthy string such as `"not-a-date"` is preserved unchanged.
- A whitespace string such as `" "` is truthy and is preserved unchanged.
- A timestamp with microseconds or a different timezone offset is kept exactly as supplied.

No parsing or timezone normalization is performed.

## Return Value

Returns a plain `dict` containing the six metadata fields.

Example shape:

```python
{
    "schema_version": "1.0",
    "project_name": "my-project",
    "created_at": "2024-01-01T12:34:56+00:00",
    "updated_at": "2024-01-01T12:34:56+00:00",
    "node_count": 42,
    "generator": "ftl-reasons/0.40.0",
}
```

The caller must handle several things themselves:

- `generator` may be `"ftl-reasons/unknown"` if package version lookup fails.
- `created_at` may be any caller-supplied truthy string, not necessarily valid ISO 8601.
- The returned dictionary is mutable; callers can modify it without affecting module state.
- If the caller passes wrong runtime types, those wrong types are stored in the returned dictionary.

## Algorithm

1. Compute the current UTC timestamp once:

   ```python
   now = datetime.now(timezone.utc).isoformat(timespec="seconds")
   ```

   This omits microseconds and includes the UTC offset as `+00:00`.

2. Build and return a new dictionary:

   - `schema_version`: `SCHEMA_VERSION`, currently `"1.0"`
   - `project_name`: the input value
   - `created_at`: `created_at` if truthy, otherwise `now`
   - `updated_at`: `now`
   - `node_count`: the input value
   - `generator`: the result of `_get_generator()`

3. `_get_generator()` attempts to read the installed `ftl-reasons` package version via `importlib.metadata.version("ftl-reasons")`. If that raises any `Exception`, it falls back to `"unknown"` and returns a generator string like `"ftl-reasons/unknown"`.

## Side Effects

No application state is mutated.

The function does perform a package metadata lookup through `_get_generator()`. That may read installed distribution metadata from the environment, but it does not write files or modify persistent state.

`build_meta` does not cache `_get_generator()`; each call can perform the version lookup again.

## Error Handling

`build_meta` has no explicit validation and no error-return path. It either returns a metadata dictionary or raises normally propagated exceptions.

Key details:

- Invalid `project_name`, negative `node_count`, malformed `created_at`, and empty `project_name` do not raise.
- `_get_generator()` catches `Exception` around the `importlib.metadata.version()` call and returns `"ftl-reasons/unknown"` on failure.
- `_get_generator()` does not catch `BaseException` such as `KeyboardInterrupt` or `SystemExit`.
- If a caller passes an object whose truthiness check raises, that exception propagates.
- If `datetime.now(...)` or timestamp formatting fails, that exception would propagate, though this is practically unusual.
- The function never returns an error dictionary.

## Usage Patterns

Typical new metadata creation:

```python
meta = build_meta("my-project", node_count=42)
```

Typical update/preservation flow:

```python
old_meta = load_existing_metadata()

meta = build_meta(
    project_name=old_meta.get("project_name", ""),
    node_count=len(nodes),
    created_at=old_meta.get("created_at", ""),
)
```

In that pattern, the caller preserves the original `created_at` while allowing `updated_at` to refresh to the current time.

Caller obligations:

- Pass a valid ISO timestamp for `created_at` if timestamp correctness matters.
- Preserve and re-pass the original truthy `created_at` during updates.
- Validate or normalize `project_name` and `node_count` upstream if needed.
- Treat `generator` as possibly unknown.
- Ensure the returned dictionary is JSON/DB-serializable; `build_meta` does not enforce that.

## Dependencies

Internal dependencies:

- `SCHEMA_VERSION` from the same module
- `_get_generator()` from the same module

External/standard-library dependencies:

- `datetime.datetime`
- `datetime.timezone`
- `importlib.metadata.version`, imported lazily inside `_get_generator`

The package metadata key used is exactly `"ftl-reasons"`.

## Assumptions Not Enforced by Type Hints

- `project_name: str` does not enforce non-empty or trimmed strings.
- `node_count: int` does not enforce non-negative values.
- `created_at: str` does not enforce ISO 8601 formatting.
- `created_at: str` does not enforce timezone consistency with `updated_at`.
- The code assumes `created_at or now` is the desired rule, so `""` and `None` both mean “use current time.”
- The code assumes callers will preserve an existing `created_at` themselves during updates.
- The code assumes `updated_at` does not need to be chronologically compared with `created_at`; it just stamps the current time.

## Topics to Explore

- [file] `tests/test_metadata.py` — Covers expected metadata keys, defaults, and generated timestamp behavior.
- [file] `tests/test_metadata_extended.py` — Likely exercises edge cases such as preserved `created_at` and generator fallback behavior.
- [function] `reasons/metadata.py:_get_generator` — Resolves the installed `ftl-reasons` version and swallows version-lookup failures.
- [general] `metadata-schema-versioning` — How `SCHEMA_VERSION` is consumed by JSON, Markdown, SQLite, and PostgreSQL export/storage paths.
- [general] `timestamp-preservation` — How callers pass existing `created_at` through updates and whether timezone/format normalization is required elsewhere.

## Beliefs

- `build-meta-returns-six-fields` — `build_meta` returns a dict with exactly `schema_version`, `project_name`, `created_at`, `updated_at`, `node_count`, and `generator`.
- `build-meta-computes-now-once` — `build_meta` computes the current UTC timestamp once per call and uses it for `updated_at` and for `created_at` only when `created_at` is falsy.
- `build-meta-preserves-truthy-created-at` — `build_meta` copies any truthy `created_at` value unchanged and replaces only falsy values with the current timestamp.
- `build-meta-does-not-validate-inputs` — `build_meta` performs no runtime validation of `project_name`, `node_count`, or `created_at`.
- `build-meta-generator-fallback-is-unknown` — `_get_generator` catches `Exception` from `importlib.metadata.version` and returns `ftl-reasons/unknown` on failure.
