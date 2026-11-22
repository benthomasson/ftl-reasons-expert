---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Map each install extra to the flags it can enable and the provider paths it unlocks.

The observations do **not** contain enough wiring to produce a fully verified “extra → CLI flag → provider path” matrix. The only authoritative extra definitions are in `pyproject.toml:14-24`; the CLI parser definitions and optional-import gating code were not captured (`all_cli_argument_definitions` returned zero matches, and `optional_dependency_gating` returned zero matches). The CLI snippets only show first 500 of 4,015 lines and no `argparse` setup.

So the table below separates **confirmed dependencies** from **observed or inferred runtime wiring**.

| Extra | Declared dependencies (`pyproject.toml`) | Observed / inferred flag or runtime switch | Provider path / entry point | Confidence |
|---|---|---|---|---|
| `test` | `pytest`, `pytest-cov`, `pyyaml` (`pyproject.toml:15`) | Not an app flag. Docs use `uv run --extra test pytest tests/ -v` (`CLAUDE.md:86`, `README.md:191`) | Test runner / `tests/` directory via `[tool.pytest.ini_options]` (`pyproject.toml:33-34`) | Confirmed as test-only extra |
| `pg` | `psycopg[binary]>=3.1` (`pyproject.toml:16`) | `args.pg` is checked in CLI functions; `REASONS_PG_CONNINFO` env var is also checked. See `cmd_mark_duplicate`, `cmd_mark_superseded`, `cmd_defeat_justification`, `cmd_defeat_with_scope`, `cmd_migrate_defeaters`, `cmd_classify_defeaters` in `reasons/cli.py` | Likely `reasons/pg.py` PostgreSQL backend. However, captured `reasons/pg.py` imports do **not** show `psycopg`, so the actual import/gating is not visible | Partial: `--pg` is observed; psycopg wiring is missing |
| `test-pg` | `psycopg[binary]>=3.1`, `pytest`, `pytest-cov` (`pyproject.toml:17`) | Same runtime path as `pg`; intended for PostgreSQL tests | `reasons/pg.py` plus test suite | Partial, same caveat as `pg` |
| `cluster` | `sentence-transformers>=2.2`, `scikit-learn>=1.2` (`pyproject.toml:18`) | No `cluster`-related CLI flag or gating observed | Likely clustering / embedding path, but no provider module or flag is confirmed | Unverified |
| `mcp` | `mcp>=1.0` (`pyproject.toml:19`) | No CLI flag observed. Console script is defined as `reasons-mcp = "reasons.mcp_server:main"` (`pyproject.toml:28`) | `reasons.mcp_server:main` MCP server entry point | Partial: entry point confirmed, import gating missing |
| `api` | `langchain-anthropic>=0.3` (`pyproject.toml:20`) | No provider flag observed | Likely Anthropic via LangChain; not confirmed | Unverified |
| `vertex` | `langchain-google-vertexai>=2.0` (`pyproject.toml:21`) | No provider flag observed | Likely Google Vertex AI via LangChain; not confirmed | Unverified |
| `langfuse` | `langfuse>=2.0` (`pyproject.toml:22`) | No provider flag observed | Likely Langfuse tracing/observability; not confirmed | Unverified |
| `openai` | `openai>=1.0` (`pyproject.toml:23`) | No provider flag observed | Likely OpenAI provider; not confirmed | Unverified |
| `llm` | `langchain-anthropic>=0.3`, `langchain-google-vertexai>=2.0`, `langfuse>=2.0` (`pyproject.toml:24`) | Composite extra; no single flag observed | Bundles Anthropic + Vertex + Langfuse. Notably, it does **not** include `openai` or `mcp` | Confirmed by dependency list |

Key observations:

- `pg` is the only extra with clear runtime check evidence: `reasons/cli.py` repeatedly checks `getattr(args, "pg", None)` and `os.environ.get("REASONS_PG_CONNINFO")`.
- Those same checks explicitly reject several commands with `"no PgApi implementation"`, e.g. `cmd_mark_duplicate`, `cmd_mark_superseded`, `cmd_defeat_justification`, and `cmd_defeat_with_scope`. This implies a `--pg` backend exists for some commands but not all.
- `reasons/storage.py:1-5` describes SQLite persistence, and `class Storage` is a SQLite implementation. The observed storage provider module is therefore SQLite-only.
- `reasons/pg.py` is likely the PostgreSQL provider path, but its captured import list contains only `json`, `re`, `collections.deque`, `datetime`, `itertools.combinations` — no `psycopg`. That means the psycopg import is either elsewhere, dynamic, or not captured.
- `mcp` unlocks the `reasons-mcp` script through `reasons.mcp_server:main`, but the entry point is declared unconditionally in `pyproject.toml:26-28`; the optional dependency only makes it runnable.
- `llm` is not a superset of every provider extra: it includes Anthropic, Vertex AI, and Langfuse, but excludes `openai`.

What is missing to complete the mapping:

1. The `argparse` parser construction in `reasons/cli.py`, especially subparsers and `add_argument` calls.
2. `_backend_kwargs(args)` in `reasons/cli.py`, which likely translates `--pg` / `--db` into backend kwargs.
3. Optional import/gating code for `psycopg`, `mcp`, `langchain-anthropic`, `langchain-google-vertexai`, `langfuse`, `openai`, `sentence-transformers`, and `scikit-learn`.
4. The rest of `reasons/pg.py` and any provider registry / backend selection logic.
5. The implementation behind `reasons.mcp_server:main`.

## Topics to Explore

- [function] `reasons/cli.py:_backend_kwargs` — Maps CLI arguments such as `--pg` and `--db` into backend kwargs; central to any extra-to-flag mapping.
- [file] `reasons/pg.py` — PostgreSQL provider implementation; confirms whether `psycopg` is imported dynamically and which commands support `--pg`.
- [file] `pyproject.toml` — Source of truth for optional dependency groups and console scripts; lines 14-24 define all extras.
- [general] `argparse-parser-definition` — Search the full `reasons/cli.py` for `add_argument` and subparser setup to verify runtime flags.
- [general] `optional-import-gating` — Find how missing optional dependencies are reported across MCP, LLM providers, clustering, and Postgres.

## Beliefs

- `llm-extra-composes-api-vertex-langfuse` — The `llm` extra installs `langchain-anthropic`, `langchain-google-vertexai`, and `langfuse`, but not `openai` or `mcp`.
- `pg-extra-corresponds-to-pg-backend-checks` — CLI commands check `args.pg` and `REASONS_PG_CONNINFO`, and several reject execution with “no PgApi implementation” when PostgreSQL is selected.
- `mcp-extra-requires-mcp-for-reasons-mcp-entrypoint` — The `reasons-mcp` console script points at `reasons.mcp_server:main`, while `mcp>=1.0` is declared only under the `mcp` extra.
- `storage-module-is-sqlite-specific` — `reasons/storage.py` implements SQLite persistence and does not contain PostgreSQL backend logic.
- `test-extra-does-not-declare-runtime-provider` — The `test` extra adds only `pytest`, `pytest-cov`, and `pyyaml`, and docs invoke it as `uv run --extra test pytest tests/ -v`.
