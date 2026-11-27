---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Compare `pyproject.toml` version against `importlib.metadata.version("ftl-reasons")` in the actual environment.

## Comparing `pyproject.toml` Version vs `importlib.metadata.version("ftl-reasons")`

### The two sides of the comparison

**1. Declared version — `pyproject.toml`**

In the `[project]` table:

```toml
[project]
name = "ftl-reasons"
version = "0.58.0"
```

This is a **static** declaration (no `dynamic = ["version"]`), so `0.58.0` is the authoritative value that a build backend will write into the installed distribution's `.dist-info/METADATA`.

**2. Runtime version — `importlib.metadata`**

- `reasons/metadata.py` (line 17) does `from importlib.metadata import version`, and `_get_generator()` calls `version("ftl-reasons")` to produce strings like `ftl-reasons/0.58.0`. It wraps the call in a bare `except Exception` and falls back to `"unknown"`.
- `reasons/cli.py` (line 13) imports it aliased: `from importlib.metadata import version as _pkg_version`.

These read the *installed* distribution's metadata (`*.dist-info/METADATA` under site-packages), **not** `pyproject.toml` directly. So the two values only agree when the package in the environment was built/installed from this `pyproject.toml`.

### What the observations can and cannot tell us

- **Can confirm:** `pyproject.toml` declares `0.58.0` for `ftl-reasons` (line 6).
- **Can confirm:** no `__version__` attribute is exported anywhere in the package (`grep __version__` returned `0` matches), so `importlib.metadata.version("ftl-reasons")` is the *only* runtime path to the version.
- **Cannot confirm:** the actual installed value in the environment. The observation set contains no `pip show ftl-reasons`, no `python -c "from importlib.metadata import version; print(version('ftl-reasons'))"`, and `.forge/report.json` only records pipeline counts (entries/proposed/accepted), not version info.

So the comparison itself requires one more step that the observation set didn't perform. It would look like:

```bash
python - <<'PY'
import tomllib, importlib.metadata
declared = tomllib.load(open("pyproject.toml","rb"))["project"]["version"]
installed = importlib.metadata.version("ftl-reasons")
print(f"declared={declared}  installed={installed}  match={declared==installed}")
PY
```

### When the two will diverge (and why it matters)

1. **Package not installed at all** (e.g., running from a source checkout without `pip install -e .`). `importlib.metadata.version("ftl-reasons")` raises `PackageNotFoundError`, `_get_generator()` catches it and emits `ftl-reasons/unknown`. A "0.58.0 vs unknown" mismatch is expected in that case, not a bug.
2. **Stale editable install.** An older `pip install -e .` from before the version bump leaves `ftl-reasons-0.57.x.dist-info` on disk; `importlib.metadata` reports the old number even though `pyproject.toml` shows `0.58.0`. Re-running `pip install -e .` fixes it.
3. **Installed from PyPI / another branch.** Whichever wheel/sdist was installed carries its own baked-in version, independent of the source tree's `pyproject.toml`.
4. **`pipx`-style isolated installs** of the `reasons` / `reasons-mcp` scripts (`[project.scripts]`) can differ from the dev venv.

### Test coverage of this invariant

`tests/test_metadata.py::TestBuildMeta::test_basic_fields` asserts only `"ftl-reasons/" in meta["generator"]` — it does **not** pin `meta["generator"]` to `ftl-reasons/0.58.0`. So a version drift between `pyproject.toml` and the installed metadata would not fail any test in this file. `TestJsonExportMeta::test_export_includes_meta` also checks only the prefix. If version alignment is an invariant you want enforced, there is currently no test asserting it.

## Topics to Explore

- [file] `pyproject.toml` — Single source of truth for `0.58.0`; check for any `dynamic = ["version"]` or `[tool.setuptools.dynamic]` block that could alter the emitted metadata (none seen in the 34 lines returned).
- [function] `reasons/metadata.py:_get_generator` — Where the runtime version string is produced and where the `except Exception → "unknown"` fallback hides install problems.
- [function] `reasons/cli.py:_pkg_version` — The other consumer of `importlib.metadata.version`; look at how it's wired into the `reasons` CLI to see what `reasons --version` actually reports.
- [general] `version-alignment-test` — Whether to add a test asserting `build_meta()["generator"] == f"ftl-reasons/{pyproject_version}"` so drift is caught in CI.
- [general] `installed-environment-probe` — Run `pip show ftl-reasons` and `python -c "from importlib.metadata import version; print(version('ftl-reasons'))"` in the actual environment to close the gap the observations left open.

## Beliefs

- `pyproject-toml-declares-version-0-58-0` — `pyproject.toml` statically pins `[project].version = "0.58.0"` for package `ftl-reasons`.
- `metadata-py-uses-importlib-metadata` — `reasons/metadata.py` reads the version exclusively via `importlib.metadata.version("ftl-reasons")`; `_get_generator` is the only caller there.
- `package-exports-no-dunder-version` — No `__version__` attribute exists anywhere in the package, so `importlib.metadata` is the sole runtime version source.
- `generator-falls-back-to-unknown` — `_get_generator()` catches all exceptions and returns `"ftl-reasons/unknown"` if `version("ftl-reasons")` raises (e.g., package not installed).
- `version-match-not-asserted-by-tests` — Existing metadata tests only assert the `"ftl-reasons/"` prefix, not an exact match against the `pyproject.toml` version.
