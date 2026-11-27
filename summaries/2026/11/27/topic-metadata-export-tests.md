---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Add coverage for installed, stale, editable, and missing distribution metadata cases.

# Add coverage for installed, stale, editable, and missing distribution metadata cases

## What this change is about

The only distribution-metadata behavior visible in the observations is in `reasons/metadata.py:_get_generator` (`reasons/metadata.py:14-21`):

```python
def _get_generator() -> str:
    try:
        from importlib.metadata import version
        v = version("ftl-reasons")
    except Exception:
        v = "unknown"
    return f"ftl-reasons/{v}"
```

This reads the **installed distribution metadata** for the `ftl-reasons` package and formats it as `ftl-reasons/<version>`. It does not read `pyproject.toml` at runtime, even though `pyproject.toml:6-7` declares:

```toml
name = "ftl-reasons"
version = "0.58.0"
```

`_get_generator()` is used by `build_meta()` (`reasons/metadata.py:24-56`), which is the single source of truth for exported/stored metadata. So the generator string appears in JSON export, markdown frontmatter, and SQLite/PostgreSQL metadata.

## Why coverage is currently missing

The existing tests only check the generator string loosely:

- `tests/test_metadata.py:TestBuildMeta.test_basic_fields` checks that `"ftl-reasons/"` appears in `meta["generator"]`.
- `tests/test_metadata_extended.py:TestBuildMetaEdgeCases.test_generator_format` checks:
  - the string starts with `"ftl-reasons/"`
  - it splits into two parts
  - the version part is non-empty

Crucially, `"ftl-reasons/unknown"` satisfies all of those checks. That means the **missing distribution** case is not actually covered: the fallback can be exercised and the existing tests still pass.

The observation searches also found no matches for `distribution`, `editable`, `stale`, `installed`, or `missing`, which confirms there are no existing tests or code comments covering these specific metadata cases.

## What the four cases mean here

Because `_get_generator()` only calls `importlib.metadata.version("ftl-reasons")`, the cases are:

### 1. Installed

The package is installed and has valid distribution metadata. `importlib.metadata.version("ftl-reasons")` returns a version such as `"0.58.0"`.

Expected current behavior:

```python
_get_generator() == "ftl-reasons/0.58.0"
```

A test should mock `importlib.metadata.version` to return a concrete version and assert the exact generator string, not just the prefix.

### 2. Stale

The installed distribution metadata says one version, while the source tree or `pyproject.toml` says another. For example, installed metadata returns `"0.57.0"` while `pyproject.toml` declares `"0.58.0"`.

Current code does **not** detect staleness. It returns whatever `importlib.metadata.version()` returns:

```python
_get_generator() == "ftl-reasons/0.57.0"
```

A test should document that behavior unless the desired change is to add staleness detection. Adding detection would require production-code changes because no current code compares installed metadata against `pyproject.toml`.

### 3. Editable

An editable install still normally exposes distribution metadata through `importlib.metadata.version()`. The generation path is the same success path as “installed.” A unit test can simulate this by mocking `version()` to return an editable-install version string, or simply a normal version string.

Important: the current code has no editable-specific branch and no detection of editable installs. If the desired coverage is “editable installs still produce a version,” a mocked success-path test is enough. If the desired coverage is “detect editable mode,” the production code does not implement that.

### 4. Missing

The distribution is not installed, so `importlib.metadata.version("ftl-reasons")` raises `PackageNotFoundError`. The broad `except Exception` at `reasons/metadata.py:19-20` converts that into `"unknown"`.

Expected current behavior:

```python
_get_generator() == "ftl-reasons/unknown"
```

This is the most important missing test because existing assertions do not distinguish `"unknown"` from a real version.

## How to add the tests

Add tests to `tests/test_metadata_extended.py:TestBuildMetaEdgeCases`, where `_get_generator` is already imported. Use `monkeypatch` to avoid depending on whether the test environment has `ftl-reasons` installed.

Example shape:

```python
from importlib.metadata import PackageNotFoundError

def test_generator_uses_installed_version(monkeypatch):
    monkeypatch.setattr("importlib.metadata.version", lambda name: "0.58.0")
    assert _get_generator() == "ftl-reasons/0.58.0"

def test_generator_uses_stale_distribution_version(monkeypatch):
    monkeypatch.setattr("importlib.metadata.version", lambda name: "0.57.0")
    assert _get_generator() == "ftl-reasons/0.57.0"

def test_generator_handles_editable_install_metadata(monkeypatch):
    monkeypatch.setattr("importlib.metadata.version", lambda name: "0.58.0")
    assert _get_generator() == "ftl-reasons/0.58.0"

def test_generator_falls_back_when_distribution_missing(monkeypatch):
    def raise_missing(name):
        raise PackageNotFoundError(name)
    monkeypatch.setattr("importlib.metadata.version", raise_missing)
    assert _get_generator() == "ftl-reasons/unknown"
```

Because `_get_generator()` imports `version` inside the function, patching `importlib.metadata.version` works for these unit tests.

## Caveats

The observations show no other distribution-metadata consumer beyond `_get_generator`. If the codebase has additional package-metadata code using `importlib.metadata.distribution()` or similar, it is not visible here. Also, “stale” and “editable” are not distinct runtime branches in the current implementation; they are variations of the success path. The tests should pin current behavior unless the production code is also being changed.

## Topics to Explore

- [function] `reasons/metadata.py:_get_generator` — The only visible path that reads installed distribution metadata
- [file] `tests/test_metadata_extended.py` — Existing generator-format tests that need exact-version and fallback coverage
- [file] `pyproject.toml` — Declares package name `ftl-reasons` and version `0.58.0`, which runtime metadata does not currently compare against
- [general] `importlib-metadata-editable-installs` — How editable installs expose package metadata and why the current code has no editable-specific branch
- [file] `reasons_lib/metadata.py` — Backward-compatibility shim that re-exports `reasons.metadata`

## Beliefs

- `generator-reads-installed-distribution-metadata` — `_get_generator` calls `importlib.metadata.version("ftl-reasons")`, not `pyproject.toml`.
- `generator-falls-back-to-unknown-on-any-exception` — `_get_generator` catches `Exception` and returns `ftl-reasons/unknown`.
- `missing-distribution-passes-existing-format-test` — `test_generator_format` accepts `ftl-reasons/unknown` because it only requires a non-empty version part.
- `stale-version-not-detected` — No code compares installed distribution version against `pyproject.toml` version, so stale metadata is returned verbatim.
- `metadata-shim-reexports-reasons-metadata` — `reasons_lib/metadata.py` re-exports `reasons.metadata` for backward compatibility.
