---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: SCHEMA_VERSION in reasons/metadata.py

## Explanation of `SCHEMA_VERSION`

`SCHEMA_VERSION` is a module-level constant in `reasons/metadata.py` that records the version of the metadata schema used by export and storage formats.

### Purpose

It centralizes the metadata schema version so JSON export, Markdown export, and SQLite/PostgreSQL storage metadata all report the same `schema_version` value.

It exists to avoid scattering the literal `"1.0"` across the codebase. In the full file, `build_meta()` embeds it into the returned metadata dictionary:

```python
"schema_version": SCHEMA_VERSION
```

It is **not** the package/application version. Package version reporting is handled separately by `_get_generator()`.

### Contract

- **Precondition:** None for reading it. The module must simply be imported successfully.
- **Postcondition:** After importing `reasons.metadata`, `reasons.metadata.SCHEMA_VERSION` is the string `"1.0"` unless another module explicitly rebinds it.
- **Invariant under normal use:** `build_meta(...)["schema_version"]` equals the current value of `reasons.metadata.SCHEMA_VERSION` at the time `build_meta()` is called.
- **Intended invariant:** The value is treated as a stable schema identifier that should only change when the metadata schema changes.
- **Python-level reality:** The binding is not enforced as immutable. Python strings are immutable, but the module global `SCHEMA_VERSION` can be reassigned.

### Parameters

`SCHEMA_VERSION` is not callable and takes no parameters.

### Return Value

Reading `SCHEMA_VERSION` yields a `str`, normally:

```python
"1.0"
```

Callers should treat it as an opaque version token unless they explicitly choose to parse it. The module does not parse or validate the string format.

### Algorithm

1. When `reasons/metadata.py` is imported, Python evaluates the assignment `SCHEMA_VERSION = "1.0"`.
2. It binds the string object `"1.0"` to the module global name `SCHEMA_VERSION`.
3. When `build_meta()` later constructs its metadata dictionary, Python looks up `SCHEMA_VERSION` from the module globals and assigns that value to the `"schema_version"` key.
4. No transformation, normalization, copying, or validation occurs.

### Side Effects

- The assignment itself has no I/O, mutation of external state, or external calls.
- Importing the module binds `SCHEMA_VERSION` into the module namespace.
- If some code rebinds `reasons.metadata.SCHEMA_VERSION`, later `build_meta()` calls in that module will use the new value. That is possible but likely unintended.
- If another module does `from reasons.metadata import SCHEMA_VERSION`, it imports the string object by name at that moment. Later rebinding `reasons.metadata.SCHEMA_VERSION` would not update the already-imported local name in that other module.

### Error Handling

`SCHEMA_VERSION` itself raises no exceptions under normal access. There is no validation, parsing, or schema compatibility check. If a caller expects a semantic version object, it must parse the string itself.

### Usage Patterns

Typical usage is indirect through `build_meta()`:

```python
from reasons.metadata import build_meta

meta = build_meta("my-project", node_count=42)
assert meta["schema_version"] == "1.0"
```

Callers may also import it directly for comparisons or tests:

```python
from reasons.metadata import SCHEMA_VERSION
```

Caller obligations:

- Treat `SCHEMA_VERSION` as read-only by convention.
- Bump it when the metadata schema changes.
- Coordinate changes with export readers, storage migrations, and tests.
- Do not confuse it with the package version from `_get_generator()`.

### Assumptions Not Enforced by the Type System

- The string format of `SCHEMA_VERSION` is not enforced. It could be `"1.0"`, `"1.0.0"`, `"v1"`, or anything else.
- The value is not automatically tied to the actual keys present in the metadata dictionary.
- No runtime schema validation confirms that consumers and producers agree on the schema.
- The module global is not protected from reassignment by `Final`, properties, or const-like enforcement.
- The intended relationship between `SCHEMA_VERSION` and the package version is documentation/convention only.

### Dependencies

`SCHEMA_VERSION` itself depends on no modules or runtime services. It is a plain string literal. The surrounding module imports `datetime`, `timezone`, and lazily imports `importlib.metadata`, but those do not affect the constant’s definition.

## Topics to Explore

- [file] `tests/test_metadata.py` — Checks the expected metadata fields and likely asserts the exact `schema_version` value.
- [function] `reasons/metadata.py:build_meta` — The primary consumer of `SCHEMA_VERSION`, embedding it into every metadata dictionary.
- [function] `reasons/metadata.py:_get_generator` — Demonstrates that package/generator version is tracked separately from metadata schema version.
- [general] `metadata-schema-evolution` — How to version, migrate, and validate metadata changes across JSON and database exports.
- [file] `reasons/metadata.py` — Full module context for how metadata is assembled for export and storage.

## Beliefs

- `schema-version-is-string-literal` — `SCHEMA_VERSION` is bound to the literal string `"1.0"` when `reasons.metadata` is imported.
- `build-meta-produces-schema-version` — `build_meta()` always includes `SCHEMA_VERSION` as the `"schema_version"` field in its returned dictionary.
- `schema-version-is-not-application-version` — The metadata schema version and the package/generator version are separate concepts and separate values.
- `schema-version-lacks-runtime-validation` — No code validates the format of `SCHEMA_VERSION` or checks it against the metadata dictionary shape.
- `schema-version-binding-is-mutable-by-convention` — Python does not prevent reassigning `reasons.metadata.SCHEMA_VERSION`, although callers should treat it as a constant.
