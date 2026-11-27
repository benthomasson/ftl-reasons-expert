---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Clarifies `importlib.metadata` name matching vs. importable modules, and the PEP 503 normalization rules that make `ftl-reasons` and `ftl_reasons` equivalent.

# How `importlib.metadata` name matching differs from importable modules

## The two namespaces involved

Python packaging distinguishes between two different "names" for the same project, and `reasons` uses both:

| Namespace | Controlled by | Example in this repo |
|---|---|---|
| **Distribution name** | `[project].name` in `pyproject.toml` | `ftl-reasons` |
| **Import package name(s)** | `[tool.setuptools].packages` in `pyproject.toml` | `reasons`, `reasons_lib` |

In `pyproject.toml` you can see all three facts together: `name = "ftl-reasons"`, `version = "0.58.0"`, and `packages = ["reasons", "reasons_lib"]`. Nothing enforces that these are the same string — and here they are deliberately different.

Consequences that show up in the code:

- **`pip install ftl-reasons`** registers a distribution named `ftl-reasons`.
- **`import reasons`** and **`import reasons_lib`** resolve to packages on `sys.path`; `reasons_lib/metadata.py:1` is literally just `from reasons.metadata import *  # noqa: F401,F403`, so the two import names are effectively aliases for the same module tree.
- **There is no `ftl_reasons` or `ftl-reasons` module.** `importlib.metadata.version("ftl-reasons")` works because the *distribution* is installed; `import ftl_reasons` would raise `ModuleNotFoundError`. Conversely, `importlib.metadata.version("reasons")` would raise `PackageNotFoundError` because `reasons` is a package name, not a distribution name.

This is the exact trap the code in `reasons/metadata.py` avoids by hard-coding the distribution name from `pyproject.toml`, not the module name.

## Where the lookup happens

Two call sites reference the distribution name directly:

- `reasons/metadata.py:17-18` — inside `_get_generator()`:
  ```python
  from importlib.metadata import version
  v = version("ftl-reasons")
  ```
- `reasons/cli.py:13` imports `version as _pkg_version`, and `reasons/cli.py:3057` uses it:
  ```python
  parser.add_argument("--version", action="version", version=f"%(prog)s {_pkg_version('ftl-reasons')}")
  ```

Both pass the literal string `"ftl-reasons"`. The generator string in `reasons/metadata.py` (`return f"ftl-reasons/{v}"`) and the CLI `--version` output are both anchored to that name.

## PEP 503 normalization: why `ftl-reasons` == `ftl_reasons`

`importlib.metadata.version(name)` does **not** perform a byte-for-byte dict lookup. It normalizes the argument using the PEP 503 rules before matching against installed distribution names:

```
re.sub(r"[-_.]+", "-", name).lower()
```

Every run of one or more `-`, `_`, or `.` collapses to a single `-`, and the whole string is lowercased. So all of these refer to the *same* distribution:

- `ftl-reasons`
- `ftl_reasons`
- `ftl.reasons`
- `FTL-Reasons`
- `ftl__reasons` (double underscore collapses)
- `ftl---reasons` (triple hyphen collapses)

That is precisely why `version("ftl-reasons")` and `version("ftl_reasons")` return the same `0.58.0` value even though only one of those spellings appears in `pyproject.toml`. The stdlib does the normalization *at lookup time*; there is nothing in the `reasons` codebase that implements it.

(Newer Pythons additionally apply PEP 685 for extras, but distribution root-name lookup is still governed by the PEP 503 regex above.)

## What happens when the distribution is not installed

`_get_generator()` in `reasons/metadata.py` wraps the lookup in `try/except Exception:` and returns `"unknown"` on failure:

```python
try:
    from importlib.metadata import version
    v = version("ftl-reasons")
except Exception:
    v = "unknown"
return f"ftl-reasons/{v}"
```

`importlib.metadata.version()` raises `PackageNotFoundError` when the distribution isn't discoverable — which is common when running directly from a checkout without `pip install -e .`. The `except` swallows all exceptions (not just `PackageNotFoundError`), so the generator falls back to `ftl-reasons/unknown` rather than crashing an export. The test at `tests/test_metadata_extended.py` (`test_generator_format`) only asserts the *happy* path — it checks `gen.startswith("ftl-reasons/")` and that the version segment is non-empty, but it does not exercise the `unknown` fallback. That branch is untested by the visible observations.

By contrast, the CLI's `_pkg_version('ftl-reasons')` at `reasons/cli.py:3057` is **not** guarded — if the distribution isn't installed, `reasons --version` will raise rather than print. That asymmetry is worth knowing.

## Where normalization is *not* happening

The `find_normalization_logic` search returned 94 matches, but none of them implement PEP 503. They are unrelated concerns:

- `reasons/api.py` `_normalize_json` / `_normalize_markdown` — normalize *belief files* for import, not distribution names.
- `reasons/build_wiki.py:49` `re.sub(r'[^a-z0-9]+', '-', label.lower()).strip('-')` — slugifies wiki labels, which is closer to a PEP 503 regex but is used for page filenames.
- `reasons/api.py` `canonical_id` — deduplication of *nodes*, not packages.

Also note: `packages_distributions()` — which would map `reasons` → `["ftl-reasons"]` — was searched for but found no matches. The codebase does **not** ask "which distribution provides the `reasons` module?"; it asserts `"ftl-reasons"` as a compile-time constant. That's fine as long as `pyproject.toml` never renames the distribution, but it means the coupling between `pyproject.toml` `name` and `reasons/metadata.py:18` / `reasons/cli.py:3057` is implicit and must be kept in sync manually.

## What the observations don't tell us

The exact PEP 503 regex lives in CPython's `importlib.metadata` / `packaging` sources, not in this repo — so from the observations we can infer the *behavior* (hyphen/underscore equivalence) but not cite the in-repo implementation. To verify end-to-end, you'd want to run `version("ftl_reasons")` against an installed wheel and confirm it returns `0.58.0`, and to check `tests/` for any existing test that exercises the `"unknown"` fallback in `_get_generator()`.

## Topics to Explore

- [file] `pyproject.toml` — The single source of truth for the `ftl-reasons` distribution name and the `reasons`/`reasons_lib` package layout; changing `name` requires touching `reasons/metadata.py:18` and `reasons/cli.py:3057`.
- [function] `reasons/metadata.py:_get_generator` — The only place the distribution name is normalized/fallback-guarded in library code; study why `except Exception` is broader than `PackageNotFoundError`.
- [function] `reasons/cli.py:main` — Around line 3057, where `--version` calls `_pkg_version('ftl-reasons')` without the fallback that `_get_generator()` provides.
- [general] `pep-503-normalization` — Verify the regex `re.sub(r"[-_.]+", "-", name).lower()` against `packaging.utils.canonicalize_name`, and test `version("FTL_Reasons")` to see normalization in action.
- [repo] `importlib-metadata-backport` — If the project ever needs Python <3.8 or custom distribution discovery, `packages_distributions()` would let it discover `reasons → ftl-reasons` instead of hard-coding the string.

## Beliefs

- `distribution-name-is-independent-of-package-name` — The distribution `ftl-reasons` provides the import packages `reasons` and `reasons_lib`; there is no importable module named `ftl_reasons`, `ftl-reasons`, or `ftl.reasons`.
- `importlib-metadata-normalizes-via-pep-503` — `version("ftl-reasons")` and `version("ftl_reasons")` return the same value because `importlib.metadata` canonicalizes names by collapsing `[-_.]+` to `-` and lowercasing before lookup.
- `get-generator-falls-back-to-unknown` — `_get_generator()` in `reasons/metadata.py` catches all exceptions from `version("ftl-reasons")` and returns `ftl-reasons/unknown` when the distribution isn't installed.
- `cli-version-has-no-fallback` — `reasons/cli.py:3057` calls `_pkg_version('ftl-reasons')` directly, so `reasons --version` raises if the distribution is not installed, unlike `_get_generator()`.
- `distribution-name-is-hardcoded` — The string `"ftl-reasons"` is duplicated in `reasons/metadata.py:18` and `reasons/cli.py:3057` rather than discovered via `packages_distributions()`, so renaming the project in `pyproject.toml` requires editing both sites.
