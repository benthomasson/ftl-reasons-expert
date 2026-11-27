---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Understand exactly when `importlib.metadata.version` raises vs. returns, especially in editable installs and zipapps.

# When `importlib.metadata.version()` raises vs. returns

## What `version()` actually does

`importlib.metadata.version("ftl-reasons")` is a thin wrapper around `Distribution.from_name("ftl-reasons").version`. The lookup is **not** based on importing `reasons`. Instead, `MetadataPathFinder` walks **`sys.path` in order** and, for each entry, looks for a matching distribution metadata directory or file:

- `<path>/ftl_reasons-<ver>.dist-info/` (wheel-style, created by pip and modern `pip install -e .`)
- `<path>/ftl_reasons.egg-info/` (or a single `.egg-info` file — classic `setup.py` / `setup.py develop` style)
- The same names **inside a zip archive** found on `sys.path` (`.zip`, `.egg`, `.whl`, or a `.pyz` — Python's `zipimport` allows `MetadataPathFinder` to walk the archive via `zipfile.Path`)

Name normalization follows PEP 503, so `ftl-reasons` and `ftl_reasons` match the same distribution declared in `pyproject.toml` (`name = "ftl-reasons"`).

## Raises `PackageNotFoundError` when

`PackageNotFoundError` (defined as a subclass of `ModuleNotFoundError`, hence also of `ImportError`) is raised **iff no matching distribution metadata is discoverable on `sys.path` at call time**. It has nothing to do with whether `import reasons` succeeds. Common causes:

1. **Running from a source checkout without ever installing.** `python -m reasons.cli --help` on a fresh clone where `pip install -e .` hasn't been run → raises.
2. **A zipapp that bundles source code but not `.dist-info`.** Tools like `python -m zipapp`, `shiv`, or `pex` can exclude metadata to save size; Python adds `app.pyz` to `sys.path[0]` automatically, but if there's no `.dist-info` inside, discovery fails.
3. **Frozen binaries** (PyInstaller/cx_Freeze/Nuitka) that don't call `copy_metadata("ftl-reasons")` in a hook. Different packaging mechanism, same failure.

Rarely, a corrupt `METADATA` file can raise `UnicodeDecodeError`/`ValueError` — but `PackageNotFoundError` is the only failure you should design for.

It **does not** validate the version against `git describe`, `__version__`, or anything else. It just reads the string from `METADATA:Version`. So even in a PEP 660 editable install, if you bump `pyproject.toml`'s `version` without re-running `pip install -e .`, `version()` returns the stale string already in `site-packages/.../ftl_reasons-0.58.0.dist-info/METADATA`.

## Editable installs specifically

| Setup | Does `version("ftl-reasons")` work? |
|---|---|
| `pip install -e .` (PEP 660, setuptools ≥ 64) | ✅ Yes — pip still writes a `.dist-info` dir into `site-packages`. `version()` returns the declared string (currently `"0.58.0"`, per `pyproject.toml`). |
| `python setup.py develop` (legacy) | ✅ Yes — the `.egg-info` dir lives next to the source, and its parent dir is on `sys.path` via `.egg-link`/`.pth`. |
| Source checkout, no install at all | ❌ `PackageNotFoundError` |

So the typical "editable install fails" story is really **"source tree, never installed"** — the dev forgot `pip install -e .`. `pyproject.toml` declares a **static** `version = "0.58.0"` (not `dynamic = ["version"]` with `setuptools-scm`), so even editable installs report the literal 0.58.0 from `pyproject.toml` at install time.

## Zipapps specifically

A `.pyz` only resolves if:
1. The archive is on `sys.path` (usually true when launched as `python app.pyz` — Python inserts it at `sys.path[0]`), **and**
2. A matching `.dist-info` (or `.egg-info`) is actually stored inside the archive.

If you build a zipapp with `python -m zipapp src -o reasons.pyz` from a bare source tree, there is no `.dist-info` inside → raise. If you use `shiv`/`pex` with `--include-dist-info` (or the equivalent), `zipimport` lets `MetadataPathFinder` see it and `version()` returns.

---

## What this codebase actually does

### `reasons/metadata.py:15–20` — defensive, never crashes

```python
def _get_generator() -> str:
    try:
        from importlib.metadata import version
        v = version("ftl-reasons")
    except Exception:
        v = "unknown"
    return f"ftl-reasons/{v}"
```

- Catch is `except Exception`, so it swallows `PackageNotFoundError` *and* any `ValueError`/`UnicodeDecodeError` from corrupt metadata.
- Fallback is `"unknown"`, so generator strings on exports degrade gracefully: `"ftl-reasons/unknown"`.
- The re-export shim `reasons_lib/metadata.py` (`from reasons.metadata import *`) forwards `build_meta` and `SCHEMA_VERSION`; `_get_generator` isn't re-exported because of the leading underscore, but the shim doesn't affect the fallback.
- The test in `tests/test_metadata_extended.py` (`test_generator_format`) does **not** distinguish installed vs. not installed — `"unknown"` is truthy, `len(parts) == 2` holds for `"ftl-reasons/unknown"`, so the test passes in either environment. `tests/test_metadata.py:test_basic_fields` likewise only asserts `"ftl-reasons/" in ...`.

### `reasons/cli.py:13, 3057` — no fallback, hard crash if not installed

```python
from importlib.metadata import version as _pkg_version
...
parser.add_argument("--version", action="version",
                    version=f"%(prog)s {_pkg_version('ftl-reasons')}")
```

- The call is inside the f-string, evaluated when `parser.add_argument(...)` runs — i.e. **eagerly during parser construction**, not lazily when `--version` is passed. So any CLI invocation (`--help`, subcommand, etc.) hits it.
- No try/except. If `ftl-reasons` isn't discoverable, `PackageNotFoundError` propagates and the CLI dies before argparse can even print usage.
- This is asymmetric with `reasons/metadata.py`: exports degrade, but the CLI has a hard dependency on install metadata. In practice this is masked because the test suite runs under an installed distribution (editable or wheel).

### `find_editable_and_zipapp_handling` returned zero matches

The codebase contains **no** `sys.path` manipulation, no `__file__`-based fallback, no `sys.frozen` check, no `_MEIPASS` check, and no explicit `PackageNotFoundError` handling (only the broad `except Exception` in `reasons/metadata.py`). So there is no edit-distribution walker like some projects (`_find_egginfo`, `importlib_metadata` backport for older Pythons, or `packaging.version.Version` parsing from `__file__`).

### `pyproject.toml`

- `name = "ftl-reasons"` and `version = "0.58.0"` — both static.
- `build-backend = "setuptools.build_meta"` — so `pip install -e .` produces a proper `.dist-info` (PEP 660), meaning the CLI works *if installed*, fails otherwise.
- `[project.scripts]` provides `reasons = "reasons.cli:main"` — no `[project.entry-points."zipapp"]` or build config for a `.pyz`; zipapp packaging is out of scope for the project.

### Bottom line

- **`version()` raises `PackageNotFoundError`** whenever no `.dist-info`/`.egg-info` matching `ftl-reasons` is on `sys.path` — which is "source tree without install" or "zipapp without embedded dist-info."
- **It returns a string** (never `None`) whenever such metadata is discovered — including most PEP 660 editable installs and zipapps built with `--include-dist-info`.
- **`reasons/metadata.py`** hides all of this behind `"unknown"`.
- **`reasons/cli.py:3057`** does not, and will crash the whole CLI on an uninstalled source checkout.
- **No test** in the observed files exercises the "not installed" path (or a zipapp at all).
- The observations **do not include** the actual editable/zipapp test matrix, `.pyz` build scripts, or any `PackageNotFoundError` test — those would be needed to verify the CLI's crash behavior in CI.

---

## Topics to Explore

- [file] `reasons/cli.py` — Confirm whether `add_argument(... _pkg_version('ftl-reasons') ...)` on line 3057 is inside an eagerly-called `_build_parser()`; the crash depends on parser construction timing.
- [file] `reasons/metadata.py` — Inspect `_get_generator` (lines 15–20) to see the broad `except Exception` fallback and whether narrowing it would be safer.
- [file] `pyproject.toml` — Check the static `version = "0.58.0"` vs. switching to `dynamic = ["version"]` with `setuptools-scm` for editable installs.
- [general] `editable-install-behavior` — Run `python -c "from importlib.metadata import version; print(version('ftl-reasons'))"` in three environments: fresh clone, `pip install -e .`, and a built wheel — compare results.
- [general] `zipapp-packaging-strategy` — Whether the project (or a downstream consumer) ships a `.pyz`; if so, confirm the build includes `ftl_reasons-0.58.0.dist-info/` inside the archive.

## Beliefs

- `metadata-fallback-hides-not-found` — `_get_generator` in `reasons/metadata.py` (lines 15–20) catches all exceptions from `version("ftl-reasons")` and substitutes `"unknown"`, so exports never fail on a missing distribution.
- `cli-version-has-no-fallback` — `reasons/cli.py:3057` calls `_pkg_version('ftl-reasons')` without a try/except inside a `parser.add_argument` call, so the CLI crashes at parser-construction time if the distribution metadata isn't discoverable.
- `version-resolution-is-sys-path-not-import` — `importlib.metadata.version("ftl-reasons")` resolves the version by scanning `sys.path` for `.dist-info`/`.egg-info`, independently of whether `import reasons` succeeds.
- `hardcoded-version-in-pyproject` — `pyproject.toml` declares `version = "0.58.0"` statically, so any successful install (editable or wheel) reports that literal string regardless of git state until re-installed.
- `no-editable-or-zipapp-special-handling` — The `editable|zipapp|sys.path|__file__` search returned zero matches, confirming the codebase contains no explicit handling for either installation mode.
