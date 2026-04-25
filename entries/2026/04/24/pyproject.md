# File: pyproject.toml

**Date:** 2026-04-24
**Time:** 16:44



# `pyproject.toml` — Project Metadata and Build Configuration

## Purpose

This is the single source of truth for how `ftl-reasons` is packaged, installed, and discovered by Python tooling. It replaces the older `setup.py`/`setup.cfg` pattern and is consumed by `uv`, `pip`, `setuptools`, and `pytest`. It defines the project's identity (name, version, license), its dependency contract (what it needs at build time vs. runtime vs. test time), and how the `reasons` CLI entry point maps to code.

## Key Components

### `[build-system]`
Declares setuptools ≥68.0 as the build backend via PEP 517. This tells `uv`/`pip` how to build a wheel or sdist. The minimum version 68.0 is significant — that's when setuptools gained stable support for PEP 621 declarative metadata in `pyproject.toml` (removing the need for `setup.cfg`).

### `[project]`
PEP 621 metadata. Notable details:

- **`name = "ftl-reasons"`** — the installable package name. The `ftl-` prefix is a namespace convention (likely "faster than light" or a team/org prefix); the importable code is `reasons_lib`.
- **`version = "0.18.0"`** — single-sourced version. No dynamic version detection; bumped manually here.
- **`requires-python = ">=3.10"`** — floor of Python 3.10, which is when `match`/`case` and `ParamSpec` landed. The codebase likely uses 3.10+ syntax.
- **`dependencies = []`** — **zero runtime dependencies**. The entire library runs on the stdlib alone (SQLite via `sqlite3`, dataclasses, etc.). This is a deliberate architectural choice: the TMS engine has no external coupling.

### `[project.optional-dependencies]`
Only `pytest` is needed, and only for testing. There's no dev extras group — no linters, formatters, or type checkers declared here (those may live in CI config or developer convention).

### `[project.scripts]`
```toml
reasons = "reasons_lib.cli:main"
```
This creates the `reasons` executable on install. Running `reasons status` or `uv run reasons add ...` invokes `reasons_lib.cli:main` with `sys.argv`. This is the bridge between the shell commands documented in CLAUDE.md and the Python code.

### `[tool.setuptools]`
Explicitly lists `reasons_lib` as the only package to include. This prevents setuptools auto-discovery from accidentally packaging `tests/` or other top-level directories into the distribution.

### `[tool.pytest.ini_options]`
Points pytest at `tests/` so you can run `pytest` from the repo root without `-k` or path arguments.

## Patterns

- **Zero-dependency library**: Everything is stdlib. This is the "vendorless" pattern — maximizes portability and minimizes supply chain risk. Appropriate for a TMS that's fundamentally a graph algorithm over SQLite.
- **Optional test deps**: The `[test]` extra keeps pytest out of production installs. Install with `pip install ftl-reasons[test]` or `uv run --extra test pytest`.
- **Explicit package list over auto-discovery**: Prevents accidental inclusion of test fixtures or scripts in the built wheel.
- **PEP 621 declarative metadata**: No `setup.py` execution at install time — everything is static and parseable.

## Dependencies

**Build-time**: `setuptools >= 68.0` (PEP 517 build backend)
**Runtime**: None
**Test-time**: `pytest` (optional extra)
**What depends on it**: `uv.lock` is generated from this file and pins exact versions. CI workflows in `.github/workflows/` consume it via `uv run`. The `reasons` CLI entry point wires to `reasons_lib/cli.py:main`.

## Flow

There's no runtime control flow — this is declarative configuration. The "execution" happens when tooling reads it:

1. `uv sync` / `pip install -e .` reads `[build-system]` → invokes setuptools → installs `reasons_lib` + creates `reasons` script shim
2. `uv run reasons ...` resolves the entry point → calls `reasons_lib.cli:main()`
3. `uv run --extra test pytest` installs pytest into the venv → pytest reads `[tool.pytest.ini_options]` → discovers tests in `tests/`

## Invariants

- The installable package name (`ftl-reasons`) and the importable package name (`reasons_lib`) are intentionally different. Don't confuse them.
- Version must be bumped here manually — there's no `__version__` import or dynamic version plugin.
- `dependencies = []` is load-bearing: if a runtime dependency is ever added, it must go here, not just in a requirements file.

## Error Handling

Not applicable — `pyproject.toml` is declarative. Malformed TOML or invalid metadata surfaces as build/install errors from setuptools or uv, not from the project code itself.

---

## Topics to Explore

- [file] `reasons_lib/cli.py` — The `main` function that the `reasons` entry point dispatches to; understanding arg parsing and subcommand routing
- [file] `uv.lock` — The locked dependency graph generated from this file; shows exact resolved versions including transitive test deps
- [file] `.github/workflows/` — How CI consumes this config to run tests and (potentially) publish
- [general] `zero-dependency-design` — Why the project chose to avoid all runtime dependencies and how that shapes the SQLite-based storage layer
- [function] `reasons_lib/network.py:Network` — The core class that the CLI entry point operates on; the heart of what this package actually ships

## Beliefs

- `zero-runtime-deps` — `ftl-reasons` has no runtime dependencies; the entire library runs on Python's stdlib alone
- `entry-point-mapping` — The `reasons` CLI command maps directly to `reasons_lib.cli:main` and is the only registered script entry point
- `version-is-manual` — The version (`0.18.0`) is statically declared in `pyproject.toml` with no dynamic version plugin or `__version__` import
- `package-name-split` — The pip-installable name is `ftl-reasons` but the importable Python package is `reasons_lib`; these are deliberately decoupled
- `python-310-floor` — The project requires Python 3.10+, establishing the minimum language features available throughout the codebase

