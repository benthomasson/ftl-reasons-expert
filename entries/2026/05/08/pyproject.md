# File: pyproject.toml

**Date:** 2026-05-08
**Time:** 14:06



# `pyproject.toml` — Project Metadata and Build Configuration

## Purpose

This is the single source of truth for how `ftl-reasons` is packaged, installed, and configured as a Python project. It defines the package identity (name, version, license), declares dependencies, registers the `reasons` CLI entry point, and configures tooling (setuptools, pytest). Every `pip install`, `uv sync`, or `python -m build` invocation reads this file to know what to do.

## Key Components

### `[build-system]`
Declares setuptools (≥68.0) as the build backend via PEP 517. This means the project uses the traditional setuptools machinery rather than newer backends like hatch or flit. The `build-backend = "setuptools.build_meta"` line is what `pip` and `build` use to locate the backend entry point.

### `[project]`
PEP 621 standardized metadata:
- **`name = "ftl-reasons"`** — the installable package name (what you'd `pip install`).
- **`version = "0.38.0"`** — statically declared, not dynamic. This is bumped manually.
- **`requires-python = ">=3.10"`** — sets the floor. The venv shown in the repo tree uses Python 3.14, so the project spans 3.10–3.14.
- **`dependencies = []`** — the core library has **zero runtime dependencies**. This is a deliberate design choice: `reasons_lib` is pure Python with no mandatory third-party packages. All heavy dependencies are optional.

### `[project.optional-dependencies]`
Four extras, each gating a feature boundary:

| Extra | Packages | Purpose |
|-------|----------|---------|
| `test` | pytest, pytest-cov | Basic test suite |
| `pg` | psycopg\[binary\]≥3.1 | PostgreSQL storage backend (`reasons_lib/pg.py`) |
| `test-pg` | psycopg + pytest + pytest-cov | Tests that hit a real Postgres instance |
| `cluster` | sentence-transformers≥2.2, scikit-learn≥1.2 | Semantic clustering of beliefs (`reasons_lib/cluster.py`) |

The `test-pg` extra is a superset combining `pg` and `test` — it exists so CI can install everything needed for Postgres integration tests in one `pip install -e ".[test-pg]"`.

### `[project.scripts]`
```toml
reasons = "reasons_lib.cli:main"
```
This registers `reasons` as a console entry point. After `pip install`, the `reasons` command invokes `reasons_lib.cli:main`. This is the primary interface for the TMS — `reasons status`, `reasons retract`, `reasons export`, etc.

### `[tool.setuptools]`
```toml
packages = ["reasons_lib"]
```
Explicitly lists the single package to include. This overrides setuptools' auto-discovery, ensuring only `reasons_lib/` is packaged — not `tests/`, `reviews/`, or other top-level directories.

### `[tool.pytest.ini_options]`
```toml
testpaths = ["tests"]
```
Tells pytest to only collect from the `tests/` directory, avoiding accidental collection from `reviews/` or other directories that might contain files matching `test_*.py` patterns.

## Patterns

- **Zero-dependency core**: The empty `dependencies = []` means the core TMS (network, storage, retraction, nogood detection) runs with nothing but the standard library. This is the "small kernel" pattern — optional features like Postgres, clustering, and ML are added via extras.
- **Feature-gated extras**: Each optional dependency group maps 1:1 to a specific module (`pg` → `pg.py`, `cluster` → `cluster.py`). This keeps install size minimal for users who only need the SQLite-based TMS.
- **Static metadata**: Version is hardcoded rather than dynamic (no `dynamic = ["version"]`). Simple but requires manual bumps.

## Dependencies

**What it depends on** (build-time): setuptools ≥68.0.

**What depends on it**: The "Imported By" list is misleading — those are setuptools/build internals that *parse* `pyproject.toml` generically during installation. They don't depend on this file specifically; they process any project's `pyproject.toml`. The real consumers are:
- `pip install` / `uv sync` — reads this to resolve and install the package
- `python -m build` — reads this to produce sdist/wheel
- `pytest` — reads `[tool.pytest.ini_options]` for configuration
- `uv.lock` — the lockfile in the repo is generated from this file's dependency declarations

## Flow

1. A developer runs `pip install -e ".[test]"` (or `uv sync`).
2. The build frontend reads `[build-system]` to bootstrap setuptools.
3. Setuptools reads `[project]` for metadata and `[tool.setuptools]` for package discovery.
4. The `reasons` console script is registered, pointing to `reasons_lib.cli:main`.
5. When a user runs `reasons status`, the OS looks up the entry point, imports `reasons_lib.cli`, and calls `main()`.

## Invariants

- The core package must remain installable with **no third-party dependencies**. Adding anything to `dependencies = []` would break this contract.
- Only `reasons_lib/` is packaged — `tests/`, `entries/`, `reviews/`, and other directories are excluded from distribution.
- Python ≥3.10 is required. Code in `reasons_lib/` can use 3.10+ features (match statements, union type syntax with `|`, etc.).

## Error Handling

`pyproject.toml` is declarative — it doesn't handle errors itself. Errors surface through tooling:
- If `requires-python` doesn't match the active interpreter, pip/uv will refuse to install.
- If an optional extra's package can't be resolved, the install fails with a dependency resolution error.
- If `reasons_lib.cli:main` doesn't exist at install time, the entry point registration succeeds but the `reasons` command will fail at runtime with an `ImportError`.

## Topics to Explore

- [file] `reasons_lib/cli.py` — The `main()` function registered as the `reasons` entry point; this is where all CLI subcommands are defined and dispatched
- [file] `reasons_lib/pg.py` — The PostgreSQL storage backend gated behind the `pg` optional dependency
- [file] `reasons_lib/cluster.py` — Semantic clustering module that requires sentence-transformers and scikit-learn
- [file] `uv.lock` — The resolved lockfile generated from this pyproject.toml; shows exact pinned versions of all transitive dependencies
- [general] `entry-point-resolution` — How setuptools console_scripts work under the hood: script wrappers, `importlib.metadata`, and the `pkg_resources` fallback path

## Beliefs

- `ftl-reasons-zero-runtime-deps` — The core `reasons_lib` package has zero mandatory runtime dependencies; all third-party packages are behind optional extras
- `reasons-cli-entry-point` — The `reasons` CLI command is a setuptools console_scripts entry point that calls `reasons_lib.cli:main`
- `setuptools-explicit-packages` — Only `reasons_lib/` is included in the distribution; `tests/`, `reviews/`, and other directories are explicitly excluded via `[tool.setuptools] packages`
- `python-version-floor-3-10` — The project requires Python ≥3.10, allowing use of match statements, union types, and other 3.10+ syntax throughout `reasons_lib`

