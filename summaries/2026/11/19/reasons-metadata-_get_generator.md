---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: _get_generator in reasons/metadata.py

## Purpose

`_get_generator()` produces the metadata `"generator"` string used by export/storage paths. It identifies the producing package and version, for example:

```python
"ftl-reasons/0.40.0"
```

It exists so `build_meta()` and downstream JSON/Markdown/SQLite/PostgreSQL metadata all use one source of truth for the generator identifier instead of hard-coding the name/version in multiple places.

The leading underscore marks it as an internal helper, not part of the public API.

## Contract

- **Preconditions:** None. The function takes no arguments and does not require caller setup.
- **Postconditions:** Returns a string beginning with `"ftl-reasons/"`.
- **Normal success:** Returns `f"ftl-reasons/{version}"`, where `version` comes from `importlib.metadata.version("ftl-reasons")`.
- **Normal failure:** If importing or calling `version()` raises any `Exception`, it returns `"ftl-reasons/unknown"`.
- **Invariant:** The returned string always has the literal prefix `"ftl-reasons/"`.
- **No validation:** The function does not validate that the resolved version is PEP 440-compliant, non-empty, or consistent with the running source tree.

## Parameters

None.

There are no argument-level edge cases. All edge cases are environment-dependent:

- The `ftl-reasons` distribution metadata may not be installed.
- The package may be importable under a different name, e.g. `reasons`, while the distribution name is `ftl-reasons`.
- The environment may contain multiple versions/distributions or stale editable-install metadata.
- `importlib.metadata.version()` may raise `PackageNotFoundError` or another exception.
- `importlib.metadata` itself may fail to import in an unusual environment.

## Return Value

Returns a `str`.

Typical values:

```python
"ftl-reasons/0.40.0"
```

Fallback value:

```python
"ftl-reasons/unknown"
```

The caller must treat this as an opaque generator identifier. It should not assume the version segment is parseable as SemVer/PEP 440 or that it matches the currently executing source code. In particular, `"unknown"` is a valid fallback and must be handled if the caller displays or stores the value.

## Algorithm

1. Enter a `try` block.
2. Lazily import `version` from the standard-library module `importlib.metadata`.
3. Call `version("ftl-reasons")`.
4. Assign the result to `v`.
5. If any `Exception` occurs during the import or the version lookup, assign `v = "unknown"`.
6. Return an f-string: `f"ftl-reasons/{v}"`.

The `version` import is inside the function, so it does not happen at module import time. This makes the module import slightly cheaper and defers any import-related failure until `_get_generator()` is actually called.

## Side Effects

- May import `importlib.metadata.version` into the function’s local scope.
- May perform filesystem/package-metadata I/O through `importlib.metadata.version()`, which searches installed distribution metadata on `sys.path`.
- Does not mutate global state, write files, log, cache, or persist anything.
- Does not cache the version between calls. Each invocation performs a fresh lookup.

## Error Handling

The function deliberately catches `Exception` broadly:

- Import failure for `importlib.metadata.version` is swallowed.
- `PackageNotFoundError` from `version("ftl-reasons")` is swallowed.
- Any other `Exception` raised during the version lookup is swallowed and replaced with `"unknown"`.

It does **not** catch `BaseException` subclasses such as `KeyboardInterrupt` or `SystemExit`.

This means the caller generally does not need to handle normal lookup failures. However, the broad catch also hides programming/environment errors such as a mistyped distribution name or corrupted package metadata, so diagnostics are not surfaced here.

## Usage Patterns

The primary caller is `build_meta()`:

```python
return {
    ...
    "generator": _get_generator(),
}
```

So every call to `build_meta()` resolves the generator string again. Tests may call `_get_generator()` directly to verify the real-version path and fallback path, often by monkeypatching `importlib.metadata.version` or simulating package-not-found behavior.

Caller obligations:

- Do not assume a real version is available.
- Treat `"ftl-reasons/unknown"` as a valid fallback.
- Do not parse the result unless you explicitly handle non-version values like `"unknown"`.

## Dependencies

- Standard library: `importlib.metadata.version`.
- Distribution metadata for the distribution named `"ftl-reasons"`.
- Indirectly, the module `reasons/metadata.py`, whose `build_meta()` function calls this helper.

## Assumptions Not Enforced by the Type System

- The installed distribution name is exactly `"ftl-reasons"`, not `"reasons"` and not some other packaging variant.
- The running environment actually has distribution metadata available, not merely the source files.
- The version string returned by `importlib.metadata.version()` is suitable for direct inclusion in the generator string.
- `_get_generator()` does not try to reconcile the installed metadata version with the version of the currently imported source code.
- A broad `Exception` fallback is acceptable; the function does not distinguish between “package not installed” and “something unexpected failed during metadata lookup.”

## Topics to Explore

- [file] `tests/test_metadata.py` — Shows the expected behavior of `_get_generator` and `build_meta`, including likely fallback coverage.
- [function] `reasons/metadata.py:build_meta` — Demonstrates how `_get_generator()` is consumed in the exported metadata dictionary.
- [general] `importlib.metadata.version` — Explains how Python resolves installed distribution metadata and why `PackageNotFoundError` can occur.
- [general] `ftl-reasons-packaging-metadata` — The distribution must be installed with the name `ftl-reasons` for the real version path to work.
- [general] `editable-install-versioning` — Important because source checkout versions and installed metadata versions can diverge.

## Beliefs

- `get-generator-output-always-has-ftl-reasons-prefix` — Every return value begins with the literal string `"ftl-reasons/"`.
- `get-generator-uses-distribution-name-ftl-reasons` — The version lookup queries the distribution name `"ftl-reasons"`, not the import package name `"reasons"`.
- `get-generator-swallows-normal-exceptions` — Any `Exception` from importing or calling `version` is caught and converted to the value `"unknown"`.
- `get-generator-does-not-cache-version` — Each invocation performs a fresh `importlib.metadata.version()` lookup.
- `build-meta-populates-generator-via-get-generator` — `build_meta()` calls `_get_generator()` when constructing the `"generator"` metadata field.
