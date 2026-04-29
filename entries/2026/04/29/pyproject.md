# File: pyproject.toml

**Date:** 2026-04-29
**Time:** 17:11



# `pyproject.toml` — Project Definition and Build Configuration

## Purpose

This is the single source of truth for the `ftl-reasons` package metadata, build system, dependencies, and tooling configuration. It replaces the legacy `setup.py` / `setup.cfg` approach with the PEP 621 declarative format. Every tool in the Python ecosystem — `pip`, `uv`, `setuptools`, `pytest` — reads this file to understand how to build, install, test, and distribute the package.

## Key Components

### `[build-system]`

Declares setuptools (≥68.0) as the build backend via PEP 517. This means `pip install .` or `uv sync` will invoke `setuptools.build_meta` to produce a wheel. The version floor of 68.0 is significant — that's the release where setuptools gained full PEP 621 (`[project]` table) support, so this project can't be built with older setuptools.

### `[project]`

Core metadata:

- **`name = "ftl-reasons"`** — the distribution name (what you `pip install`).
- **`version = "0.22.0"`** — static version, manually bumped. No dynamic versioning plugin.
- **`requires-python = ">=3.10"`** — hard floor. Pattern matching, `match/case`, `ParamSpec`, and `|` union types are all available.
- **`dependencies = []`** — the library has **zero runtime dependencies**. The entire `reasons_lib` package runs on the standard library alone (SQLite via `sqlite3`, JSON, argparse, etc.). This is a deliberate design choice — it makes the tool trivially installable anywhere Python 3.10+ exists.

### `[project.optional-dependencies]`

Three extras, which form a clear dependency lattice:

| Extra      | Adds                              | Purpose                          |
|------------|-----------------------------------|----------------------------------|
| `test`     | `pytest`, `pytest-cov`            | Run the SQLite-based test suite  |
| `pg`       | `psycopg[binary]>=3.1`           | PostgreSQL storage backend       |
| `test-pg`  | `psycopg[binary]`, pytest, cov   | Run tests including PG tests     |

`test-pg` is the union of `test` and `pg` — it doesn't reference them via extras cross-references, it just duplicates the deps. Slightly redundant but explicit.

### `[project.scripts]`

```toml
reasons = "reasons_lib.cli:main"
```

This registers a single console entry point: the `reasons` CLI. After `pip install ftl-reasons`, typing `reasons` on the command line invokes `reasons_lib.cli:main()`. This is the primary user interface for the entire system — belief management, import/export, derivation, staleness checking.

### `[tool.setuptools]`

```toml
packages = ["reasons_lib"]
```

Explicit package discovery — only `reasons_lib/` is included in the distribution. The `tests/`, `reviews/`, `entries/`, and `.code-expert/` directories are excluded from the wheel. This prevents test fixtures and knowledge-base artifacts from shipping to end users.

### `[tool.pytest.ini_options]`

```toml
testpaths = ["tests"]
```

Tells pytest to only collect from `tests/`. Without this, pytest's default collection would also descend into `reasons_lib/` looking for `test_*.py` files, and potentially into `entries/` or `reviews/`.

## Patterns

- **Zero-dependency core**: The empty `dependencies = []` is the most important architectural signal in this file. It means `reasons_lib` is a self-contained library that uses only `sqlite3`, `json`, `argparse`, and other stdlib modules. Optional backends (PostgreSQL) are opt-in via extras.
- **Extras as feature flags**: The `pg` / `test-pg` split follows the common pattern of making heavy or platform-specific dependencies opt-in. Code in `reasons_lib/pg.py` presumably guards its `import psycopg` behind a try/except or is only invoked when the extra is installed.
- **Single entry point**: One CLI (`reasons`) is the sole user-facing command. All subcommands (`status`, `explain`, `export`, `derive`, etc.) are handled internally by `reasons_lib.cli:main`.

## Dependencies

**What this file depends on** (build-time): `setuptools>=68.0`

**What depends on this file**:
- `pip install .` / `uv sync` — reads `[build-system]` and `[project]`
- `pytest` — reads `[tool.pytest.ini_options]`
- `uv.lock` — generated from this file's dependency declarations
- CI workflows in `.github/workflows/` — likely reference the extras (`pip install .[test]`)

## Flow

There is no runtime control flow — this is a declarative configuration file. The "execution" is:

1. A build frontend (`pip`, `uv`, `build`) reads `[build-system]` and bootstraps setuptools.
2. Setuptools reads `[project]` for metadata and `[tool.setuptools]` for package discovery.
3. A wheel is produced containing only `reasons_lib/`.
4. On install, the `reasons` script shim is generated pointing at `reasons_lib.cli:main`.

## Invariants

- The package **must** be built with setuptools ≥ 68.0 (PEP 621 support).
- Python ≥ 3.10 is required at install time — pip/uv will refuse to install on 3.9 or older.
- Only `reasons_lib/` is packaged — test and documentation directories are excluded.
- The `reasons` CLI entry point must resolve to a callable `main()` in `reasons_lib.cli`.

## Error Handling

Not applicable in the traditional sense. Build errors manifest as:
- `setuptools` version too old → build fails with missing `[project]` table support.
- Missing Python version → resolver refuses to install.
- Missing optional dependency → `ImportError` at runtime when accessing PG features without `pip install .[pg]`.

---

## Topics to Explore

- [file] `reasons_lib/cli.py` — The `main()` function that the `reasons` entry point dispatches to; understand all available subcommands
- [file] `reasons_lib/pg.py` — How the PostgreSQL backend is conditionally loaded when the `pg` extra is installed
- [file] `uv.lock` — The resolved dependency lock file generated from this pyproject.toml; shows exact pinned versions
- [file] `.github/workflows/` — CI configuration that consumes the test extras and runs the test suite
- [general] `zero-dependency-design` — Why the project chose zero runtime dependencies and how `sqlite3` from stdlib serves as the default storage engine

## Beliefs

- `zero-runtime-dependencies` — `ftl-reasons` declares no runtime dependencies; the entire `reasons_lib` package runs on Python's standard library alone
- `single-cli-entry-point` — The only registered console script is `reasons`, pointing to `reasons_lib.cli:main`; there are no other executables
- `python-310-floor` — The project requires Python ≥ 3.10 and will not install on older interpreters
- `only-reasons-lib-packaged` — Setuptools is configured to include only the `reasons_lib` package in distributions; tests, entries, and reviews are excluded from wheels

