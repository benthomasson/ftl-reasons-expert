---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Explains how Python resolves installed distribution metadata and why `PackageNotFoundError` can occur.

# How Python Resolves Installed Distribution Metadata (and Where `PackageNotFoundError` Comes From)

## What `importlib.metadata.version()` actually does

When the code calls `version("ftl-reasons")` (in `reasons/metadata.py:18` and via `_pkg_version('ftl-reasons')` in `reasons/cli.py:3057`), it is **not** doing anything package-specific. `importlib.metadata` performs a filesystem scan for distribution metadata:

1. **Canonicalize the name.** The argument is run through the PEP 503 normalization rules: lowercased, and `.` / `_` / `-` collapsed to `-`. So `ftl-reasons`, `ftl_reasons`, and `FTL.Reasons` all resolve to the same key.
2. **Walk `sys.path` in order.** For each entry (site-packages, the venv, any `PYTHONPATH` entries, and — for editable installs — the project root via a `*.pth` or `__editable__*.pth` stub), it looks for directories with a `.dist-info` or `.egg-info` suffix whose basename (before the version) canonicalizes to the requested name.
3. **Parse `METADATA`.** Inside the matched `*.dist-info` directory it reads the `METADATA` file (RFC 822-style headers) and extracts the `Name:` and `Version:` fields. `version()` returns the `Version:` value verbatim; `metadata["Name"]` returns the `Name:` value.
4. **Fail on first miss.** If no `sys.path` entry contains a matching `.dist-info`, `distribution()` raises `importlib.metadata.PackageNotFoundError`.

Concretely, in this repo the metadata for this distribution is written by setuptools (build backend `setuptools.build_meta`, `requires = ["setuptools>=68.0"]` at the top of `pyproject.toml`) into something like `ftl_reasons-0.58.0.dist-info/METADATA`, derived from `[project] name = "ftl-reasons"` and `version = "0.58.0"`.

## Why `PackageNotFoundError` can occur

`PackageNotFoundError` is raised for a very narrow reason: **the interpreter running the code cannot find a distribution whose canonical name matches the one requested.** It does *not* mean the module is unimportable, and it does *not* mean the code is broken. Specific causes relevant here:

- **Source checkout, no install.** Running `python -m pytest` or `python reasons/cli.py --version` directly from a clone (with `reasons/` on `sys.path` implicitly via cwd) will happily import `reasons.metadata`, but there is no `.dist-info` directory anywhere on `sys.path`, so `version("ftl-reasons")` fails. This is by far the most common case.
- **Different interpreter / venv.** The package may be installed for a system Python but the tests or CLI are run under a project-local venv (or vice versa). `sys.path` is per-interpreter, so metadata resolution is per-interpreter.
- **Name mismatch.** The lookup uses the string `"ftl-reasons"` in both call sites (`reasons/metadata.py:18`, `reasons/cli.py:3057`). Because of PEP 503 canonicalization, an install of `ftl_reasons`, `ftl.reasons`, or `Ftl-Reasons` will match. An install under a *different* name (e.g., someone re-tagged the wheel as `reasons`) will not.
- **Partial / legacy editable installs.** Some editable-install modes only drop a `__editable__*` path stub without a `.dist-info`, or an old `setup.py`-style install can leave behind a stale `.egg-info` that doesn't correspond to the current version or name.
- **Zip / `pipx`-style isolated installs.** If `sys.path` contains a zipimporter, `.dist-info` directories inside the zip are still discoverable by `importlib.metadata` on modern Python, but an unusual distribution layout (e.g., a `*.pyz`) can defeat it.
- **Multiple distributions installed.** Lookup returns the first matching one encountered along `sys.path`, so a stale copy earlier on the path can shadow the intended one.

## How this repo deals with it (and where it doesn't)

The two call sites handle the exception very differently, which is the most important thing to understand here:

- **`reasons/metadata.py:15-20` (`_get_generator`)** wraps the call in a broad `try/except Exception` and falls back to `v = "unknown"`. The function is documented to return `'ftl-reasons/0.40.0'`, and the export/import system (`build_meta`, markdown frontmatter, SQLite `network_meta`) is designed so that a missing version never blocks an export. The test `test_generator_format` in `tests/test_metadata_extended.py` only asserts the prefix and that the second segment is non-empty — so `"ftl-reasons/unknown"` passes as well.
- **`reasons/cli.py:13` + `reasons/cli.py:3057`** imports `version as _pkg_version` at module level and hands the result directly to `argparse.add_argument(..., action="version", version=f"%(prog)s {_pkg_version('ftl-reasons')}")`. There is **no `try/except` around this call**, and it runs during parser construction — i.e., *before* any subcommand runs. If the package is not installed, `PackageNotFoundError` would propagate out of `main()` when the parser is built, breaking the entire CLI, not just `--version`. This is the asymmetry between the two call sites.

**A note on the observations:** `find_package_not_found_error_usages` returned **0 matches** for `PackageNotFoundError`. The exception is therefore not handled by name anywhere — it is either swallowed by the blanket `except Exception` in `_get_generator` or left uncaught in `cli.py`. Anyone hardending this should search for `importlib.metadata` and `except Exception` rather than `PackageNotFoundError`.

**What the observations do not show:** the full body of `reasons/cli.py`, so I cannot confirm whether the module-level import at line 13 or the `add_argument` call at line 3057 is wrapped in a broader import guard elsewhere. Similarly, `tests/test_cli.py:test_version` at line 966 is shown only by name — its assertions would reveal whether the test suite requires an installed distribution (which would make `PackageNotFoundError` a test-environment failure rather than a user-facing one).

## Topics to Explore

- [file] `reasons/cli.py` — Full parser construction path; verify whether `_pkg_version('ftl-reasons')` at line 3057 has any surrounding guard, and confirm the failure mode when metadata is absent.
- [function] `reasons/metadata.py:_get_generator` — The sole place version resolution is shielded from `PackageNotFoundError`; check the exception granularity of `except Exception`.
- [function] `reasons/metadata.py:build_meta` — Caller of `_get_generator`; verify that downstream consumers (JSON export, markdown frontmatter, SQLite `network_meta`) treat `"ftl-reasons/unknown"` the same as a real version string.
- [file] `tests/test_cli.py` — Inspect `test_version` (~line 966) to see whether the suite assumes `ftl-reasons` is installed, and how it is skipped or set up.
- [general] `editable-install-and-packaging` — How setuptools `build_meta` emits `.dist-info`/`__editable__` artifacts for `pyproject.toml` at the repo root, since that layout determines whether `version("ftl-reasons")` succeeds in dev environments.

## Beliefs

- `distribution-lookup-requires-a-dist-info-on-sys-path` — `importlib.metadata.version("ftl-reasons")` returns a value only when some `sys.path` entry contains a canonical-name-matching `*.dist-info` or `*.egg-info` directory.
- `get-generator-swallows-package-not-found-error` — `reasons/metadata.py:_get_generator` (lines 15–20) catches every exception from `version()` and returns `"ftl-reasons/unknown"`.
- `cli-version-flag-has-no-metadata-fallback` — `reasons/cli.py:3057` calls `_pkg_version('ftl-reasons')` unguarded inside `argparse.add_argument(..., action="version")`, so a missing distribution aborts parser construction rather than only failing `--version`.
- `pyproject-declares-distribution-name-ftl-reasons` — `pyproject.toml` `[project].name = "ftl-reasons"` and `version = "0.58.0"` are the values setuptools writes into the `METADATA` that `version()` returns.
- `metadata-tests-accept-unknown-version` — `tests/test_metadata_extended.py:test_generator_format` asserts only a `ftl-reasons/` prefix and a non-empty version segment, so the `"unknown"` fallback passes the test.
- `no-explicit-package-not-found-handling` — A search for `PackageNotFoundError` across the codebase returned zero matches; the exception is handled only incidentally via `except Exception` in `reasons/metadata.py`.
