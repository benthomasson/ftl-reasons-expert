---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: pyproject.toml

# pyproject.toml

## Purpose

This is the single source of build, packaging, and test-runner configuration for the `ftl-reasons` project. It owns four responsibilities:

1. **Build system declaration** — tells pip/PEP 517 frontends to use `setuptools>=68.0` via `setuptools.build_meta`.
2. **Distribution metadata** — name, version, license, Python floor, README reference.
3. **Dependency contracts** — the core runtime dependency set (deliberately empty) and a matrix of optional extras that map 1:1 to subsystem capabilities (`pg`, `cluster`, `mcp`, LLM providers, `langfuse`).
4. **Entry points and test discovery** — exposes the `reasons` and `reasons-mcp` console scripts and pins pytest to the `tests/` directory.

It is not imported by any Python module; its consumers are the packaging toolchain, pip, CI installs, and pytest.

## Key Components

### `[build-system]`
```toml
requires = ["setuptools>=68.0"]
build-backend = "setuptools.build_meta"
```
PEP 518/517 declaration. Requires setuptools ≥ 68 (needed for the `[project]` PEP 621 table and modern `license = "MIT"` SPDX-string form). No setuptools plugins (wheel, etc.) are pinned — they come in transitively.

### `[project]`
| Field | Value | Contract |
|---|---|---|
| `name` | `ftl-reasons` | The pip-installable distribution name — **distinct** from the importable package names (`reasons`, `reasons_lib`). |
| `version` | `0.58.0` | Static single-source-of-truth for version; no `dynamic = ["version"]` and no `version = {attr = ...}`. Bumping the release means editing this line. |
| `requires-python` | `>=3.10` | Hard floor. Any use of `X | Y` unions or `match` statement in the codebase is a bet this is enforced by the installer. |
| `dependencies` | `[]` | **Zero core dependencies** — base install is pure stdlib. Every third-party import in `reasons/` must be guarded / lazy so that `pip install ftl-reasons` works on a bare interpreter. |
| `license` | `"MIT"` | SPDX identifier string form (PEP 639), not the legacy `{text = "MIT"}` table. |

### `[project.optional-dependencies]`
Nine named extras. They are **capability bundles**, not separate distributions:

| Extra | Pulls | Backed subsystem (in `reasons/`) |
|---|---|---|
| `test` | `pytest`, `pytest-cov`, `pyyaml` | Test harness; `pyyaml` is used by the fixture/config loader in tests (see `tests/conftest.py`). |
| `pg` | `psycopg[binary]>=3.1` | `reasons/pg.py` — Postgres storage/network backend. |
| `test-pg` | `psycopg[binary]>=3.1` + `pytest` + `pytest-cov` | Convenience composite: run the PG-dependent tests (`test_pg.py`, `test_pg_dispatch.py`) without installing two extras. |
| `cluster` | `sentence-transformers>=2.2`, `scikit-learn>=1.2` | `reasons/cluster.py` — embedding + clustering for topic detection. |
| `mcp` | `mcp>=1.0` | `reasons/mcp_server.py` and `reasons/mcp_client.py` — Model Context Protocol. |
| `api` | `langchain-anthropic>=0.3` | Anthropic provider path in `reasons/llm.py` / `reasons/ask.py`. |
| `vertex` | `langchain-google-vertexai>=2.0` | Google Vertex provider path. |
| `langfuse` | `langfuse>=2.0` | Tracing/observability wrapper used by `reasons/llm.py`. |
| `openai` | `openai>=1.0` | Standalone OpenAI SDK — **not** part of the `llm` composite; used where the code talks to OpenAI directly rather than via LangChain (e.g. `reasons/import_agent.py`, `reasons/hf.py`). |
| `llm` | `langchain-anthropic`, `langchain-google-vertexai`, `langfuse` | The "turn on real LLM providers" bundle: `api` ∪ `vertex` ∪ `langfuse`. Intentionally excludes `openai`. |

### `[project.scripts]`
```toml
reasons      = "reasons.cli:main"
reasons-mcp  = "reasons.mcp_server:main"
```
Two console-script entry points. They install executable shims into `PATH` that call `reasons.cli.main` and `reasons.mcp_server.main` respectively. Both targets must be zero-argument callables (they typically do `sys.exit(main())` internally). Note that the primary CLI lives in `reasons/`, **not** `reasons_lib/`, so `reasons_lib` is a library-only mirror.

### `[tool.setuptools]`
```toml
packages = ["reasons", "reasons_lib"]
```
Explicit package list (equivalent to `setuptools.find_packages` filtered down). This is a **flat layout** — both packages ship at the top level of `site-packages`. Any new top-level package directory added to the repo is **silently excluded** from the wheel until it is added to this list.

### `[tool.pytest.ini_options]`
```toml
testpaths = ["tests"]
```
`pytest` with no args only walks `tests/`. `tests/__init__.py` present means tests are a package; `conftest.py` is at `tests/conftest.py`.

## Patterns

- **Extras as feature flags.** Each subsystem that needs a third-party library is opt-in via `pip install ftl-reasons[pg]`, `[cluster]`, `[mcp]`, or `[llm]`. The base install is zero-dependency, so the codebase must use *deferred imports* (import inside the function that needs the library) rather than module-level imports for anything in an extra. This is the central architectural constraint imposed by this file.
- **Composite extras.** `test-pg` and `llm` are unions of other groups. This is not a native TOML-inheritance mechanism — the lists are duplicated literally, so changing `psycopg` in `pg` requires changing it in `test-pg` too.
- **Static version.** No `dynamic = ["version"]` → the version string in the file is authoritative and must be bumped manually (or by a release script) on each release; there is no `reasons.__version__` source of truth to sync.
- **Two-parallel-packages layout.** `reasons` and `reasons_lib` ship together as one distribution — this is unusual and suggests either (a) a vendored/mirrored copy, or (b) a transitional rename where one is the "real" package and the other is a shim. Either way, both are exported to users and both must be kept importable.
- **Explicit package listing** rather than `find = { where = ["src"] }` — this is the "no accidental packaging" idiom.

## Dependencies

**What this file depends on (build-time):**
- `setuptools>=68.0` (from `[build-system].requires`).
- The string `README.md` (from `readme = "README.md"`) — must exist at build root or the build fails.
- Each package listed must exist as a directory with `__init__.py`, or the wheel build fails.

**What depends on this file:**
- **pip / build / twine** — every install, wheel build, and upload reads this.
- **CI workflows** in `.github/workflows/` — will install using one or more extras from here.
- **pytest** — reads `[tool.pytest.ini_options].testpaths` when invoked without arguments.
- **Console-script shims** — `reasons` and `reasons-mcp` are generated from `[project.scripts]`.
- Not imported by any Python source file; there is no `import` relationship.

**Runtime dependency rules this file encodes:**
- Base runtime: Python stdlib only.
- `reasons/pg.py` → `psycopg` (extra `pg` / `test-pg`).
- `reasons/cluster.py` → `sentence-transformers`, `scikit-learn` (extra `cluster`).
- `reasons/mcp_server.py`, `reasons/mcp_client.py` → `mcp` (extra `mcp`).
- `reasons/llm.py`, `reasons/ask.py` → `langchain-anthropic` and/or `langchain-google-vertexai`, optionally `langfuse` (extras `api`, `vertex`, `langfuse`, `llm`).
- `reasons/import_agent.py` / `hf.py` → `openai` (extra `openai`).

## Flow

1. **Build**: `python -m build` reads `[build-system]` → bootstraps an isolated env with `setuptools>=68.0` → runs `setuptools.build_meta` → reads `[project]` for metadata → reads `[tool.setuptools].packages` and packages exactly `reasons/` and `reasons_lib/` → emits an sdist + wheel with two console-script shims registered from `[project.scripts]`.
2. **Install**: `pip install ftl-reasons` installs the wheel with **no** third-party deps. `pip install 'ftl-reasons[llm,pg]'` resolves the union of `langchain-anthropic`, `langchain-google-vertexai`, `langfuse`, `psycopg[binary]`. Installs generate `bin/reasons → reasons.cli:main` and `bin/reasons-mcp → reasons.mcp_server:main`.
3. **Test**: `pytest` (from repo root) reads `pyproject.toml`, sees `testpaths = ["tests"]`, and collects only within `tests/`. `pytest-cov` is available only because `test`/`test-pg` are installed in CI.
4. **Invoke**: at runtime, `reasons <subcommand>` dispatches through `reasons.cli.main` (see `tests/test_cli.py`); `reasons-mcp` starts the MCP server (`tests/test_mcp_server.py`).

## Invariants

- **Python ≥ 3.10** at install and runtime. Anything using `|` in type heads, `X | None`, `match`, or `typing.Self` is covered by this floor; dropping to 3.9 is a breaking metadata change.
- **Base install must remain import-safe with zero third-party packages.** Any module that needs an extra dep must import it lazily and raise an informative error pointing the user at the right extra (otherwise `pip install ftl-reasons` followed by `import reasons` would fail).
- **Only `reasons` and `reasons_lib` are shipped.** Adding a new top-level dir without updating `[tool.setuptools].packages` results in a wheel that omits it — a silent breakage only visible after install.
- **Two console scripts only.** `reasons` and `reasons-mcp` are the public CLI surface. Any additional entry point must be added to `[project.scripts]`.
- **The version strings must be unique per release.** `0.58.0` here must not be reused after publication.
- **`readme = "README.md"`** requires the file to exist; deleting/renaming it breaks packaging.
- **`llm` is the union of `api`, `vertex`, `langfuse`** — the three lists must stay in sync if any of them changes (no TOML-level inclusion exists).

## Error Handling

`pyproject.toml` has no runtime error path — it fails *loudly at build time*:
- Missing `README.md` → build fails ("readme file not found").
- Listed package not on disk → setuptools raises `Package 'X' not found`.
- Invalid SPDX license string → setuptools ≥ 68 will refuse to build (later versions warn; recent versions error).
- Unresolvable extra requirement → pip resolver error at install, not at import.
- Importing a subsystem without its extra → **runtime** `ModuleNotFoundError` from the deferred import in e.g. `reasons/pg.py`; whether this is wrapped in a friendlier `ImportError("install ftl-reasons[pg]")` depends on the individual module (see `tests/test_pg.py` for the expected shape).

There is no retry, fallback, or try/except behavior in this file — it is declarative configuration, and any irregularity surfaces at the packaging toolchain boundary.

## Topics to Explore

- [file] `.github/workflows/` — Which extras are installed in CI (especially whether `test-pg`/`llm` are installed on every matrix entry) determines what the CI actually exercises.
- [file] `reasons/pg.py` — Shows how the `pg` extra is imported lazily and what the "install the extra" error looks like in practice.
- [file] `reasons/llm.py` — Owns the `api` / `vertex` / `langfuse` / `openai` branching; explains why `openai` is deliberately excluded from the `llm` composite extra.
- [general] `reasons-vs-reasons_lib` — Two parallel package dirs both ship in the same wheel; worth understanding whether `reasons_lib` is a mirror, a shim, or a legacy copy (compare `reasons/merkle.py` which has no `reasons_lib` counterpart).
- [file] `tests/conftest.py` — Confirms the `pyyaml` test extra's role and how tests handle missing optional deps.

## Beliefs

- `core-dependencies-are-empty` — The base distribution declares `dependencies = []`, so `pip install ftl-reasons` pulls in zero third-party packages and all extras must be lazily imported.
- `llm-extra-excludes-openai` — The `llm` composite extra is exactly `api` ∪ `vertex` ∪ `langfuse` and deliberately does not include `openai`, which is offered separately as its own extra.
- `packages-list-requires-manual-update` — `[tool.setuptools].packages` explicitly lists `reasons` and `reasons_lib`; any new top-level package is silently excluded from wheels until added to this list.
- `version-is-static` — Package version `0.58.0` is a static string with no `dynamic` source, so it must be bumped manually in this file for every release.
- `pytest-only-collects-tests-dir` — `testpaths = ["tests"]` restricts default pytest collection to the `tests/` directory.
- `cli-entry-point-lives-in-reasons` — The `reasons` console script targets `reasons.cli:main`, not `reasons_lib.cli:main`, making `reasons/` the canonical CLI package.
