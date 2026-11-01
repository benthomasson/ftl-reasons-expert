---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: pyproject.toml

# `pyproject.toml` — Project Configuration for ftl-reasons

## 1. Purpose

This file is the **single source of truth** for the project's build identity, dependency graph, and tool configuration. It owns:

- **Package identity**: name (`ftl-reasons`), version (`0.58.0`), license (MIT), and the minimum Python interpreter (`>=3.10`).
- **Distribution packaging**: how setuptools discovers and ships code (both `reasons/` and `reasons_lib/`).
- **Optional capability tiers**: which third-party libraries are needed for PostgreSQL, LLM calls, clustering, MCP, etc.
- **CLI entry points**: the two console scripts a user invokes from the shell.
- **Test harness wiring**: where pytest looks for tests.

A developer adding a new backend, a new CLI verb, or a new test suite touches this file. It is the contract between "what this project *is*" and the Python packaging ecosystem.

## 2. Key Components

### `[build-system]`
```toml
requires = ["setuptools>=68.0"]
build-backend = "setuptools.build_meta"
```
Pins the PEP 517 build backend. `setuptools>=68.0` ensures the `pyproject.toml`-native config is fully supported (no `setup.py` fallback). Any tool that builds or installs this package (pip, Poetry, uv) will spawn an isolated environment with this requirement.

### `[project]`
| Field | Value | Significance |
|-------|-------|--------------|
| `name` | `ftl-reasons` | Distribution name on PyPI (hyphenated, as required by PEP 503). |
| `version` | `0.58.0` | Pinned manually — no `dynamic` key, so version bumps are a one-line edit. |
| `requires-python` | `>=3.10` | Hard floor. Code may use `match`, `X | Y` type unions, etc. |
| `dependencies` | `[]` | **Zero mandatory third-party deps.** The core belief-engine logic is pure-stdlib. |
| `license` | `MIT` | SPDX identifier. |

The empty `dependencies` list is the most architecturally significant line: it means `pip install ftl-reasons` gives you a fully working belief-retraction engine with no network or database access. Everything else is opt-in.

### `[project.optional-dependencies]`

These are the "extras" a user activates with `pip install ftl-reasons[pg,openai]`:

| Extra | Packages | Activates |
|-------|----------|-----------|
| `test` | `pytest`, `pytest-cov`, `pyyaml` | Local dev loop |
| `pg` | `psycopg[binary]>=3.1` | PostgreSQL persistence (`reasons/pg.py`) |
| `test-pg` | `psycopg[binary]>=3.1`, `pytest`, `pytest-cov` | Composite: PG + test (avoids listing both extras) |
| `cluster` | `sentence-transformers>=2.2`, `scikit-learn>=1.2` | Embedding-based belief clustering (`reasons/cluster.py`) |
| `mcp` | `mcp>=1.0` | Model Context Protocol server/client (`reasons/mcp_server.py`, `reasons/mcp_client.py`) |
| `api` | `langchain-anthropic>=0.3` | Anthropic/LLM integration |
| `vertex` | `langchain-google-vertexai>=2.0` | Google Vertex AI |
| `langfuse` | `langfuse>=2.0` | LLM observability/tracing |
| `openai` | `openai>=1.0` | OpenAI API |
| `llm` | `langchain-anthropic>=0.3`, `langchain-google-vertexai>=2.0`, `langfuse>=2.0` | **Meta-extra** bundling the "full LLM stack" minus `openai` |

Note: `openai` is deliberately *excluded* from the `llm` meta-extra. If you need OpenAI, you list it separately: `[llm,openai]`.

### `[project.scripts]`
```toml
reasons = "reasons.cli:main"
reasons-mcp = "reasons.mcp_server:main"
```
Two console entry points. After `pip install`, the user gets:
- `reasons` → calls `reasons.cli:main()` — the primary CLI (add beliefs, query, derive, repair, etc.).
- `reasons-mcp` → calls `reasons.mcp_server:main()` — spins up an MCP server so external agents (Claude Desktop, IDE plugins) can invoke the engine.

Both must define a `main()` callable that returns `None` (or an exit code).

### `[tool.setuptools]`
```toml
packages = ["reasons", "reasons_lib"]
```
Explicit package list. This tells setuptools to include **both** top-level packages in the wheel. This is the dual-package pattern — `reasons` is the full engine (includes `merkle.py`), while `reasons_lib` is a near-identical parallel tree (lacks `merkle.py`). Both ship in the same distribution.

### `[tool.pytest.ini_options]`
```toml
testpaths = ["tests"]
```
When a developer runs bare `pytest` from the repo root, it discovers tests only under `tests/`. No `conftest.py` at the root is needed for discovery; `tests/conftest.py` handles fixtures.

## 3. Patterns

- **Zero-core-dependency / capability-gating**: The core is stdlib-only. Every integration (DB, LLM, embeddings, MCP) is an *extra*. This keeps the attack surface minimal and lets CI test the core without installing 500 MB of ML libraries. The `llm` meta-extra is a convenience bundle, not a hard requirement.

- **Dual-package distribution**: Two parallel package trees (`reasons/`, `reasons_lib/`) ship in one distribution. This is an unusual pattern — likely `reasons_lib` is a "library-facing" API (no Merkle proofs, no internals) while `reasons` is the full implementation. Both are importable post-install. Maintainers must keep them in sync manually (no code generation evident).

- **Composite extras**: `test-pg` composes `pg` + `test` deps so CI jobs don't need `pip install ftl-reasons[pg,test]`. This is a pragmatic shortcut to reduce shell boilerplate.

- **Meta-extra (`llm`)**: A "kitchen sink" group that bundles multiple backends. Not a hard dependency on any single one — just a convenience. The `openai` exclusion is intentional (perhaps it's still experimental or has a different license/SDK pattern).

- **Explicit package list over auto-discovery**: `packages = ["reasons", "reasons_lib"]` rather than relying on setuptools' auto-detection. This prevents accidental inclusion of `tests/` or `.claude/` in the wheel.

## 4. Dependencies

### This file depends on (imports):
Nothing at runtime — it's a configuration artifact. But it *references*:
- `reasons.cli:main` — must exist and be callable.
- `reasons.mcp_server:main` — must exist and be callable.
- `reasons/` and `reasons_lib/` package directories — must exist for the wheel build.
- `README.md` — must exist (referenced by `readme` key).
- `tests/` directory — must exist (referenced by `testpaths`).

### What depends on this file:
- **CI pipelines** (`.github/workflows/`) — install with specific extras.
- **`tests/conftest.py`** — likely conditionally imports `psycopg`, `sentence_transformers`, `mcp`, etc., gated by whether the extras are installed.
- **Downstream consumers** — `pip install ftl-reasons[pg,openai]` reads this file to resolve the dependency graph.
- **Packaging tools** (build, wheel, sdist) — read `[build-system]` and `[tool.setuptools]`.

## 5. Flow

There is no runtime "execution flow" in a TOML config file. The *lifecycle* flow is:

1. **Build time**: `pip install` or `python -m build` invokes the PEP 517 frontend → isolates env → installs `setuptools>=68.0` → calls `setuptools.build_meta` → setuptools reads `[project]` + `[tool.setuptools]` → discovers `reasons/` and `reasons_lib/` → produces a `.whl` containing both packages + metadata (including the optional-dependencies table and console-script shims).
2. **Install time**: pip resolves `dependencies = []` (nothing) plus whatever extras the user requested. Creates `reasons` and `reasons-mcp` executable shims on `PATH` pointing at `reasons.cli:main` / `reasons.mcp_server:main`.
3. **Test time**: Developer runs `pytest` → reads `[tool.pytest.ini_options]` → discovers `tests/` → `conftest.py` conditionally imports optional deps.
4. **Runtime**: The CLI or MCP server starts, and user code imports from `reasons` or `reasons_lib`. Optional features (PG, LLM) are imported lazily inside their respective modules and will `ImportError` if the extra wasn't installed.

## 6. Invariants

- **`requires-python = ">=3.10"` is a hard floor.** No code path should use syntax or stdlib features from 3.9 or below. Conversely, you may assume `functools.cache`, `typing.Protocol`, structural pattern matching, etc.
- **`dependencies = []` must stay empty** (or at minimum, contain only pure-stdlib-adjacent packages). The architectural contract is that the belief engine works with zero network access. Moving a package from an extra into core `dependencies` is an API-level decision.
- **`reasons.mcp_server:main` and `reasons.cli:main` must be importable at install time** (they're referenced in the wheel metadata). If you rename or move either, the wheel build succeeds but the console script will `ImportError` at first invocation.
- **`packages` list must be exhaustive.** If a new top-level package is added (e.g., `reasons_experimental/`), it must be appended here or it won't ship in the wheel.
- **Version is not dynamic.** Every release requires a manual bump. There is no `hatch-vcs` or `setuptools-scm` integration.
- **The `llm` extra does NOT include `openai`.** If you assume `llm` gives you all LLM providers, you'll be surprised.

## 7. Error Handling

This file doesn't "handle errors" in the runtime sense, but it defines **failure modes**:

- **Missing extra at import time**: `reasons/pg.py` does `import psycopg` at module top (or inside a function). If the user hasn't installed `[pg]`, they get a raw `ImportError`. There is no graceful degradation — the module simply won't import. (You'd need to check `reasons/pg.py` to confirm whether it guards with `try/except ImportError` and raises a friendlier message.)
- **Build failure**: If `setuptools<68` is present in the build env, the PEP 621 `[project]` table is silently ignored (older setuptools doesn't understand it). The `requires = ["setuptools>=68.0"]` guard prevents this, but only if the build frontend respects PEP 517 isolation (all modern pip versions do).
- **Stale entry points**: If `reasons/cli.py` is deleted but `[project.scripts]` still references it, `pip install` succeeds (it's just metadata) but running `reasons` gives `ModuleNotFoundError`. There is no build-time validation that the dotted path resolves.

---

## Topics to Explore

- [file] `reasons/cli.py` — The `main()` entry point referenced by the `reasons` console script; shows how the CLI dispatches to subcommands (add, query, derive, repair, etc.) and how it lazily imports optional backends.
- [file] `reasons/pg.py` — The PostgreSQL persistence layer gated behind the `pg` extra; shows the error-handling pattern when `psycopg` is absent and how the storage abstraction (`reasons/storage.py`) is wired.
- [file] `reasons_lib/__init__.py` — The dual-package `reasons_lib` tree; understanding how it differs from `reasons/` (notably the absence of `merkle.py`) reveals the library-vs-engine API split.
- [general] `dual-package-sync-strategy` — How (or whether) `reasons/` and `reasons_lib/` are kept in sync. The file tree shows near-identical files; there's no visible codegen or symlink mechanism, implying manual duplication.
- [file] `.github/workflows/` — The CI pipeline(s) show which extras are installed in which job, confirming the practical "test matrix" that the `test-pg` composite extra was designed to support.

---

## Beliefs

- `core-engine-has-zero-third-party-deps` — The `reasons` package imports only Python stdlib at the module level; all external integrations (psycopg, sentence-transformers, mcp, langchain) are strictly optional extras and are not importable without explicit installation.
- `llm-extra-excludes-openai` — The `llm` optional-dependency group bundles `langchain-anthropic`, `langchain-google-vertexai`, and `langfuse` but deliberately omits `openai`, which must be installed as a separate extra.
- `reasons-contains-merkle-but-reasons-lib-does-not` — The `reasons/` package tree includes `merkle.py` (Merkle-tree integrity proofs) while the parallel `reasons_lib/` tree does not, indicating a deliberate API surface difference between the full engine and the library-facing package.
- `test-pg-composite-extra-prevents-redundant-listing` — The `test-pg` extra exists as a convenience composition of `pg` + `test` dependencies, implying that no CI job or developer workflow should need to list `[pg,test]` separately.
- `version-is-manually-pinned` — The package version (`0.58.0`) is a literal string in `pyproject.toml` with no `dynamic` declaration, meaning every release requires a manual edit and there is no git-tag-driven versioning.
