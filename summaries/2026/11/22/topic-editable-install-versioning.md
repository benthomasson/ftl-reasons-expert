---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Important because source checkout versions and installed metadata versions can diverge.

## Explanation

`reasons/metadata.py` is the metadata builder used for exports and storage. The important detail is in `_get_generator()` (`reasons/metadata.py:14-21`):

- It calls `importlib.metadata.version("ftl-reasons")` at lines 17-18.
- `importlib.metadata` reads installed distribution metadata from the active Python environment, typically from a `.dist-info` directory for `ftl-reasons`, not from the source checkout.
- If that lookup raises any exception, it falls back to `"unknown"` at lines 19-20.
- It then returns `f"ftl-reasons/{v}"` at line 21.

The source checkout version is declared separately in `pyproject.toml` line 7:

```toml
version = "0.58.0"
```

The package `reasons/__init__.py` does not expose a `__version__`; it only defines dataclasses such as `Justification`, `Node`, and `Nogood`. So there is no source-code-level version fallback independent of installed metadata.

This divergence matters because `build_meta()` (`reasons/metadata.py:24-55`) writes `_get_generator()` into the `"generator"` field at line 54. The module docstring says this metadata is used by JSON export, markdown export, and SQLite/PostgreSQL storage at lines 1-5. Therefore an export or stored record can be produced from checkout version `0.58.0` but be labeled `ftl-reasons/0.57.0` if the installed distribution metadata is stale, or `ftl-reasons/unknown` if the distribution is not installed. That is why source checkout versions and installed metadata versions diverging is important: the exported provenance can be wrong or missing.

Observation caveat: `grep_version_usage` reports zero matches, but the concrete `reasons/metadata.py` read clearly shows `importlib.metadata` usage inside `_get_generator()`. The grep result is likely incomplete or path-filtered. Also, `find_metadata_tests` reports no matching tests, so this version-resolution behavior is not covered by the observed test data.

## Topics to Explore

- [function] `reasons/metadata.py:_get_generator` — Runtime version resolution and fallback behavior for the generator string.
- [file] `pyproject.toml` — Declares the source checkout version and the package name used by `importlib.metadata.version("ftl-reasons")`.
- [file] `reasons/__init__.py` — Confirms the package does not expose `__version__` from source.
- [general] `installed-vs-checkout-version` — Compare `pyproject.toml` version against `importlib.metadata.version("ftl-reasons")` in the actual environment.
- [general] `metadata-export-tests` — Add coverage for installed, stale, editable, and missing distribution metadata cases.

## Beliefs

- `generator-version-reads-installed-metadata` — `_get_generator()` calls `importlib.metadata.version("ftl-reasons")` and does not read `pyproject.toml`.
- `pyproject-toml-owns-source-version` — `pyproject.toml` declares `version = "0.58.0"` for the `ftl-reasons` project.
- `build-meta-emits-generator-field` — `build_meta()` writes `_get_generator()` output into the `"generator"` field of the metadata dict.
- `missing-distribution-yields-unknown` — Any exception from `importlib.metadata.version("ftl-reasons")` is caught and converted to `ftl-reasons/unknown`.
- `package-init-has-no-version-attribute` — `reasons/__init__.py` defines dataclasses but does not define `__version__`.
