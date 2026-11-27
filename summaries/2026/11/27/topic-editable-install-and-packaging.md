---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: How setuptools `build_meta` emits `.dist-info`/`__editable__` artifacts for `pyproject.toml` at the repo root, since that layout determines whether `version("ftl-reasons")` succeeds in dev environments.

## How `setuptools.build_meta` emits metadata for this root layout

`pyproject.toml` sits at the repo root, so running `pip install -e .` from the repo root lets pip discover it and pass it to the declared PEP 517 backend:

- `pyproject.toml:3` declares `build-backend = "setuptools.build_meta"`.
- `pyproject.toml:6-7` declares static project metadata:
  - `name = "ftl-reasons"`
  - `version = "0.58.0"`
- `pyproject.toml:31` limits the distribution packages to:
  - `reasons`
  - `reasons_lib`

Because the version is static, `setuptools.build_meta` does not need to import `reasons` to determine `0.58.0`. It can generate metadata directly from `pyproject.toml`.

### What artifacts are emitted

For a normal wheel build, `setuptools.build_meta` emits a distribution metadata directory whose name is normalized:

```text
ftl_reasons-0.58.0.dist-info/
```

Inside it, `METADATA` contains at least:

```text
Name: ftl-reasons
Version: 0.58.0
```

For an editable install, pip uses the PEP 660 editable build path, and `setuptools.build_meta` emits the same `.dist-info` plus editable-specific artifacts in `site-packages`, such as:

```text
ftl_reasons-0.58.0.dist-info/
__editable__.ftl_reasons-0.58.0.pth
__editable___ftl_reasons_0_58_0_finder.py
```

The exact editable filenames vary by setuptools version, but the important part is that the editable install still installs a real `.dist-info` directory for the distribution. The `__editable__` finder/path hooks only make the source directories `reasons/` and `reasons_lib/` importable from the repo checkout.

### Why `version("ftl-reasons")` succeeds or fails

`importlib.metadata.version("ftl-reasons")` does **not** read `pyproject.toml`. It scans `sys.path` for installed distribution metadata, typically `.dist-info` or `.egg-info`, and normalizes the distribution name for lookup. So:

- If the project has been installed, e.g. `pip install -e .` from the repo root, the `ftl_reasons-0.58.0.dist-info` directory is present in `site-packages`, and `version("ftl-reasons")` returns `0.58.0`.
- If the code is merely run from the repo root without an editable install, there may be no matching `.dist-info` on `sys.path`. In that case `version("ftl-reasons")` raises `PackageNotFoundError`.
- The root-level `pyproject.toml` is necessary for pip to discover and build the project correctly, but the file itself is not a runtime metadata source for `importlib.metadata`.

This matters in `reasons/metadata.py:14-21`. `_get_generator` calls:

```python
from importlib.metadata import version
v = version("ftl-reasons")
```

at `reasons/metadata.py:17-18`, then catches all exceptions and falls back to `"unknown"` at `reasons/metadata.py:19-20`. Therefore:

- Installed dev environment: `build_meta()` produces `"generator": "ftl-reasons/0.58.0"`.
- Non-installed dev environment: `build_meta()` produces `"generator": "ftl-reasons/unknown"`.

`reasons/cli.py:13` also imports `importlib.metadata.version`, so it has the same dependency on an installed distribution if it calls version lookup.

One caveat: the observation `find_ftl_reasons_references` reports zero matches for `ftl-reasons`, but the displayed `pyproject.toml` and `reasons/metadata.py` clearly contain that string. The explanation above relies on the displayed file contents.

## Topics to Explore

- [file] `reasons/metadata.py` — Shows the exact `importlib.metadata.version("ftl-reasons")` fallback behavior.
- [function] `reasons/metadata.py:_get_generator` — Contains the try/except that converts missing distribution metadata into `"ftl-reasons/unknown"`.
- [file] `pyproject.toml` — Defines the build backend, static name/version, and explicit packaged directories.
- [general] `pep-660-editable-install` — Explains how `setuptools.build_meta` produces `.dist-info` plus `__editable__` finder/path artifacts.
- [file] `reasons/cli.py` — Also imports `importlib.metadata.version`, so CLI behavior may depend on editable installation.

## Beliefs

- `pyproject-declares-static-version` — `pyproject.toml` sets `name = "ftl-reasons"` and `version = "0.58.0"` statically at lines 6-7.
- `build-backend-is-setuptools-build-meta` — `pyproject.toml` line 3 selects `setuptools.build_meta` as the PEP 517 build backend.
- `version-lookup-requires-installed-metadata` — `importlib.metadata.version("ftl-reasons")` succeeds only when a matching `.dist-info` or `.egg-info` distribution is on `sys.path`; the root `pyproject.toml` alone is not sufficient.
- `metadata-helper-catches-version-errors` — `reasons/metadata.py:_get_generator` catches any exception from `version("ftl-reasons")` and returns `ftl-reasons/unknown`.
- `packages-are-explicitly-limited` — `[tool.setuptools] packages = ["reasons", "reasons_lib"]` at `pyproject.toml:31` limits the distribution to those two top-level packages.
