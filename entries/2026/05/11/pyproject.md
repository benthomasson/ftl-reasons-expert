# File: pyproject.toml

**Date:** 2026-05-11
**Time:** 12:52

## Purpose

`pyproject.toml` is the single packaging and build configuration file for the `ftl-reasons` project. It defines how the package is built, installed, and distributed — replacing the older `setup.py`/`setup.cfg` approach. It also configures pytest. This file is the authority on what the package is called, what it depends on, and what CLI entry points it exposes.

## Key Components

### Build System (lines 1–3)

```toml
[build-system]
requires = ["setuptools>=68.0"]
build-backend = "setuptools.build_meta"
```

Declares setuptools as the build backend per PEP 517. The `>=68.0` floor ensures support for the `[project]` table format (PEP 621). Any tool that builds the package (`pip install .`, `uv build`, `python -m build`) reads this to know which backend to invoke.

### Project Metadata (lines 5–12)

- **name**: `ftl-reasons` — the installable package name (what you `pip install`).
- **version**: `0.40.0` — semver, pre-1.0, signaling the API is still evolving.
- **requires-python**: `>=3.10` — uses Python 3.10+ features (match statements, union types with `|`, etc.).
- **dependencies**: `[]` — **the core library has zero runtime dependencies**. This is a deliberate design choice: the base TMS engine is pure Python with no external packages. All optional functionality (Postgres, clustering, MCP) is gated behind extras.

### Optional Dependencies (lines 14–19)

Five dependency groups, each unlocking a specific capability:

| Extra | Packages | Purpose |
|-------|----------|---------|
| `test` | pytest, pytest-cov | Run the test suite |
| `pg` | psycopg[binary]>=3.1 | PostgreSQL storage backend (`reasons_lib/pg.py`) |
| `test-pg` | psycopg + pytest | Test the Postgres backend specifically |
| `cluster` | sentence-transformers, scikit-learn | Semantic clustering of beliefs (`reasons_lib/cluster.py`) |
| `mcp` | mcp>=1.0 | Model Context Protocol client (`reasons_lib/mcp_client.py`) |

Install with e.g. `pip install "ftl-reasons[pg,cluster]"`.

### CLI Entry Point (lines 21–22)

```toml
[project.scripts]
reasons = "reasons_lib.cli:main"
```

Registers a `reasons` command that calls `reasons_lib.cli:main()`. This is how all the `reasons status`, `reasons explain`, `reasons export-markdown` commands from CLAUDE.md work.

### Setuptools Config (lines 24–25)

```toml
[tool.setuptools]
packages = ["reasons_lib"]
```

Explicitly lists the single package to include. This prevents setuptools auto-discovery from accidentally bundling `tests/`, `reviews/`, or other top-level directories into the distribution.

### Pytest Config (lines 27–28)

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
```

Tells pytest to only look in `tests/` for test files, avoiding accidental collection from `.venv/` or other directories.

## Patterns

- **Zero-dependency core**: The base package has no runtime deps. This is a common pattern for library-grade Python packages — it minimizes install footprint and avoids version conflicts. Features requiring heavy deps (ML models for clustering, async Postgres drivers) are opt-in via extras.
- **Single `pyproject.toml`**: All config (build, metadata, testing) consolidated into one file per PEP 621. No `setup.py`, `setup.cfg`, `MANIFEST.in`, or `pytest.ini`.
- **Explicit package list**: Rather than relying on `find_packages()` auto-discovery, the single package is declared explicitly — safer for a repo that mixes code with data directories.

## Dependencies

**What it depends on** (build-time): `setuptools>=68.0`. At runtime: nothing mandatory; optionally `psycopg`, `sentence-transformers`, `scikit-learn`, `mcp`.

**What depends on it**: The "Imported By" list is all setuptools/build internals and pydantic-settings — these are tools that *read* `pyproject.toml` during build or config loading, not actual code dependencies. No application code imports this file.

## Flow

1. A user runs `pip install .` or `uv sync`.
2. The build frontend reads `[build-system]` and invokes `setuptools.build_meta`.
3. Setuptools reads `[project]` for metadata, `[tool.setuptools]` for package discovery.
4. The `reasons` script entry point is registered in the environment's `bin/`.
5. Running `reasons <subcommand>` invokes `reasons_lib.cli:main()`.

## Invariants

- The core package (`reasons_lib`) must remain installable with zero dependencies. Any new import of an optional package must be behind a lazy import guarded by the corresponding extra.
- `reasons_lib` is the only distributed package — `tests/`, `reviews/`, `entries/` are never shipped.
- Python 3.10 is the minimum supported version.

## Error Handling

Not applicable — `pyproject.toml` is declarative configuration. Errors surface at build time (e.g., missing setuptools version, invalid TOML syntax) and are reported by the build frontend (pip, uv, build), not by this file.

## Topics to Explore

- [file] `reasons_lib/cli.py` — The `main()` entry point registered as the `reasons` script; defines all subcommands
- [file] `reasons_lib/pg.py` — PostgreSQL backend gated behind the `pg` extra; how optional deps are lazy-imported
- [file] `reasons_lib/cluster.py` — Semantic clustering feature requiring the heaviest optional deps (sentence-transformers, scikit-learn)
- [file] `uv.lock` — The locked dependency resolution; shows exact versions installed for all extras
- [general] `optional-dependency-gating` — How each `reasons_lib` module conditionally imports its optional packages without crashing when the extra isn't installed

## Beliefs

- `ftl-reasons-zero-runtime-deps` — The core `reasons_lib` package has no mandatory runtime dependencies; all external packages are gated behind optional extras.
- `reasons-cli-entrypoint-is-cli-main` — The `reasons` CLI command maps to `reasons_lib.cli:main`; changing that function signature or location breaks the installed command.
- `only-reasons-lib-is-distributed` — Only the `reasons_lib` package is included in built distributions; `tests/`, `entries/`, and `reviews/` are excluded by the explicit `packages = ["reasons_lib"]` declaration.
- `python-310-minimum` — The project requires Python 3.10+, which implies availability of structural pattern matching and `X | Y` union type syntax throughout the codebase.

