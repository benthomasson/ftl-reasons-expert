# File: pyproject.toml

**Date:** 2026-05-05
**Time:** 15:20



# `pyproject.toml` — Project Definition and Build Configuration

## Purpose

This is the single source of truth for how `ftl-reasons` is packaged, installed, and discovered by Python tooling. It defines the project's identity (name, version, license), its build system, its CLI entry point, and its test/optional dependency groups. Every tool in the Python ecosystem — `pip`, `uv`, `build`, `pytest` — reads this file to understand what this project is and how to work with it.

## Key Components

### `[build-system]`
Declares setuptools (≥68.0) as the build backend via PEP 517. This means `pip install .` or `python -m build` will delegate to `setuptools.build_meta` to produce wheels/sdists. Version 68+ is significant — that's when setuptools stabilized its PEP 621 `[project]` table support.

### `[project]`
PEP 621 metadata:
- **`name = "ftl-reasons"`** — the installable package name (what you `pip install`).
- **`version = "0.33.0"`** — manually maintained, no dynamic versioning.
- **`requires-python = ">=3.10"`** — floor is 3.10, and the venv is actually running 3.14, so this project spans a wide Python range.
- **`dependencies = []`** — the core library has **zero runtime dependencies**. This is a deliberate design choice: the TMS engine is pure Python with no external requirements. Everything optional (Postgres, testing) lives in extras.

### `[project.optional-dependencies]`
Three extras, two of which overlap:
- **`test`**: `pytest` + `pytest-cov` — for running the test suite against the default SQLite backend.
- **`pg`**: `psycopg[binary]>=3.1` — adds PostgreSQL storage support (`reasons_lib/pg.py`).
- **`test-pg`**: union of the above two — install this to run the full test suite including Postgres tests.

Note: `test-pg` manually lists all deps rather than referencing `test` and `pg` (which PEP 621 allows via self-referencing extras). This means adding a test dep requires updating two places.

### `[project.scripts]`
```toml
reasons = "reasons_lib.cli:main"
```
Registers the `reasons` CLI command. On install, this creates a console script that calls `reasons_lib.cli:main()`. This is the entry point for all commands: `reasons status`, `reasons explain`, `reasons retract`, etc.

### `[tool.setuptools]`
```toml
packages = ["reasons_lib"]
```
Explicitly tells setuptools which package to include — just the `reasons_lib` directory. Without this, setuptools' auto-discovery might pick up `tests/` or other directories. The `tests/` package is intentionally excluded from distribution.

### `[tool.pytest.ini_options]`
```toml
testpaths = ["tests"]
```
Directs pytest to only collect from `tests/`. This prevents it from scanning `entries/`, `reviews/`, or the venv.

## Patterns

- **Zero-dependency core**: The empty `dependencies = []` is the most important architectural signal in this file. The TMS engine runs on the stdlib alone (SQLite via `sqlite3`). This makes it embeddable anywhere without version conflicts.
- **Extras for optional backends**: Postgres support is opt-in via `pip install ftl-reasons[pg]`, following the standard pattern for alternative storage backends.
- **Flat package layout**: One top-level package (`reasons_lib`), no namespace packages, no `src/` layout. Simple and direct.

## Dependencies

**Build-time**: `setuptools>=68.0` only.

**Runtime**: None (core). Optional: `psycopg[binary]>=3.1` for Postgres.

**Dev/test**: `pytest`, `pytest-cov`.

**What depends on it**: The "Imported By" list shows `build` and `pydantic_settings` reading this file — that's the build toolchain parsing metadata, not application code importing it. `uv.lock` is generated from this file's dependency declarations.

## Flow

There's no "execution" per se — this is declarative metadata. But the key lifecycle:

1. `uv sync` / `pip install -e .` reads this file, installs the package, and creates the `reasons` console script.
2. Running `reasons <command>` invokes `reasons_lib.cli:main()`.
3. `pytest` reads `[tool.pytest.ini_options]` and collects from `tests/`.
4. `python -m build` reads `[build-system]` and produces distributable artifacts.

## Invariants

- The `reasons` CLI always resolves to `reasons_lib.cli:main` — there's no plugin system or alternate entry point.
- Only `reasons_lib/` is distributed; `tests/` is development-only.
- Python ≥3.10 is required; anything below will fail at install time.
- Core functionality must work with zero external packages installed.

## Error Handling

Not applicable — this is a declarative configuration file. Errors surface as build/install failures from the tooling that reads it (pip, setuptools, uv).

---

## Topics to Explore

- [function] `reasons_lib/cli.py:main` — The CLI entry point registered by `[project.scripts]`; understand what commands are available and how they're dispatched
- [file] `reasons_lib/pg.py` — The optional PostgreSQL backend that motivates the `pg` extra; see how it coexists with the default SQLite storage
- [file] `reasons_lib/storage.py` — The default SQLite storage layer that runs with zero dependencies; the core that `dependencies = []` enables
- [file] `uv.lock` — The resolved lockfile generated from this pyproject.toml; shows the full transitive dependency graph for all extras
- [general] `test-pg-extra-duplication` — The `test-pg` extra manually lists all deps instead of composing `test` + `pg`; a minor maintenance risk worth understanding

---

## Beliefs

- `ftl-reasons-zero-runtime-deps` — The core library declares no runtime dependencies; the TMS engine runs on the Python stdlib alone (including `sqlite3`)
- `reasons-cli-entrypoint-is-cli-main` — The `reasons` console script always resolves to `reasons_lib.cli:main()` with no plugin or alternate dispatch mechanism
- `tests-excluded-from-distribution` — Only `reasons_lib` is packaged for distribution; the `tests/` directory is excluded via explicit `[tool.setuptools] packages` configuration
- `pg-extra-required-for-postgres` — PostgreSQL support requires installing the `pg` or `test-pg` optional extra; it is not available in a bare install

