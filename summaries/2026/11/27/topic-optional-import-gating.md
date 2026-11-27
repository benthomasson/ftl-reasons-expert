---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Find how missing optional dependencies are reported across MCP, LLM providers, clustering, and Postgres.

# How Missing Optional Dependencies Are Reported

All four subsystems follow the same two-phase pattern:

1. **Lazy availability probe** — a module-level `try: import X … except ImportError:` sets a boolean flag or `None` sentinel, so importing `reasons.*` never fails on a missing optional package.
2. **Guard function that raises** — a private `_require_*()` helper raises an `ImportError` with a message that names the missing package *and* the pip extras target from `pyproject.toml`.

The slight variations (when the guard runs, whether the message references the extra or the raw package) are the interesting parts.

## MCP

Two files, two strategies.

**Client side — `reasons/mcp_client.py` (lazy, soft-fail)**
- Lines ~12–18: `try: from mcp import ClientSession, StdioServerParameters` … `except ImportError: _mcp_available = False`.
- `_require_mcp()` raises:
  > `"mcp is required for MCP server support. Install it with: pip install 'mcp>=1.0'"`
- Called only from `McpBridge.__init__`. Importing `reasons.mcp_client` or constructing `McpBridge` with no call to `connect()` still works; only actual use fails.
- Note: the message references the raw package (`mcp>=1.0`), not the extras name `ftl-reasons[mcp]`, so it drifts slightly from the cluster/pg messages.

**Server side — `reasons/mcp_server.py` (eager, hard-fail)**
- The `try: from mcp.server.fastmcp import FastMCP \n except ImportError:` at the very top immediately `raise`s:
  > `"The 'mcp' package is required for the MCP server. Install with: pip install 'ftl-reasons[mcp]'"`
- This is intentional: the `reasons-mcp` console script (`pyproject.toml` `[project.scripts]`) *cannot* function without `mcp`, so failing at import time gives a clean startup error instead of a half-broken `FastMCP` object.

## LLM Providers — `reasons/llm.py`

Four independent `try/except ImportError` blocks near the top set `_HAS_LANGCHAIN_ANTHROPIC`, `_HAS_LANGCHAIN_VERTEX`, `_HAS_LANGFUSE`, `_HAS_OPENAI`. The guards are scoped to the code path that needs the package, not to module import:

- `_invoke_api()` handles two model prefixes:
  - `api:` → `raise ImportError("langchain-anthropic is required for api: models. Install with: pip install 'ftl-reasons[api]'")`
  - `vertex:` → same shape but for `langchain-google-vertexai` / `ftl-reasons[vertex]`. Also raises a plain `RuntimeError` if `GOOGLE_CLOUD_PROJECT` is unset — a config error, not a dependency error.
- `_invoke_openai()` → `raise ImportError("openai is required for openai: models. Install with: pip install 'ftl-reasons[openai]'")`
- **Langfuse is the silent one**: `_get_langfuse_handler()` simply returns `None` when `_HAS_LANGFUSE` is `False` (or when `LANGFUSE_PUBLIC_KEY` is absent). No error is surfaced; tracing is just skipped.

Two things worth flagging:
- The `invoke_model()` docstring explicitly documents the contract: *"Raises ImportError if API dependencies are not installed (API models). Raises FileNotFoundError if the model binary is not in PATH (CLI models)."* So the LLM layer distinguishes a **Python dependency** (ImportError, has an extras fix) from a **system dependency** (FileNotFoundError for `claude`/`gemini`/`ollama` — fixed by installing a CLI, not a pip extra).
- The extras in `pyproject.toml` map 1:1 to these messages: `api`, `vertex`, `openai`, `langfuse`, plus the aggregating `llm` extra that bundles the first three.

## Clustering — `reasons/cluster.py`

- Docstring at line 6: `Requires: pip install 'ftl-reasons[cluster]'` — advertises the extra up front.
- Line 17 `except ImportError:` sets `HAS_CLUSTER_DEPS = False`; line 28 is the raise inside `_require_cluster_deps()`:
  > `"sentence-transformers and scikit-learn are required for --cluster mode. Install them with: pip install 'ftl-reasons[cluster]'"`
- Guard is invoked from **four** entry points: `ClusterCache.__init__`, `cluster_beliefs`, `cluster_beliefs_intra`, and `list_clusters`. That means any public function that touches the cache/model will fail helpfully, but importing `reasons.cluster` alone is safe.
- Message is slightly more verbose than the others because two packages are needed at once (`sentence-transformers` *and* `scikit-learn`).

## Postgres — `reasons/pg.py`

- Top of file: `try: import psycopg \n except ImportError: psycopg = None`.
- `_require_psycopg()` raises:
  > `"psycopg is required for PostgreSQL support. Install it with: pip install 'psycopg[binary]>=3.1'"`
- Guard runs in `PgApi.__init__`. So `PgApi` construction fails with a clear message; any lower-level helper (e.g. `_add_node_raw`, `add_node`) is never reached without a working connection — meaning the guard acts as a constructor-level gate rather than being sprinkled through every method.
- Unlike the others, the message names the concrete install spec (`psycopg[binary]>=3.1`) rather than the `ftl-reasons[pg]` extra — matches the pattern in `mcp_client.py` (raw package), and both differ from `cluster.py`, `mcp_server.py`, and `llm.py` (extras name).

## Summary of the pattern

| Subsystem | Where the `try/except` lives | Guard function | Raised where | Message references |
|---|---|---|---|---|
| MCP client | `mcp_client.py` module top | `_require_mcp` | `McpBridge.__init__` | raw package (`mcp>=1.0`) |
| MCP server | `mcp_server.py` module top | *(none — raises eagerly)* | import time | extras (`ftl-reasons[mcp]`) |
| LLM (Anthropic/Vertex/OpenAI) | `llm.py` module top, four flags | `_invoke_api` / `_invoke_openai` | inside each provider branch | extras (`ftl-reasons[api]`, `[vertex]`, `[openai]`) |
| LLM (Langfuse) | as above | *(none)* | never — silently skipped | n/a |
| Clustering | `cluster.py` line 17 | `_require_cluster_deps` | 4 public functions + `ClusterCache.__init__` | extras (`ftl-reasons[cluster]`) |
| Postgres | `pg.py` module top (`psycopg = None`) | `_require_psycopg` | `PgApi.__init__` | raw spec (`psycopg[binary]>=3.1`) |

## How these surface to users

`reasons/cli.py` line 1696 contains `except ImportError as e:` — almost certainly the catch site that turns these lazy raises into a single user-facing CLI error instead of a traceback. The observations don't include the surrounding block, so the exact handling (bare print vs. re-raise with exit code) isn't visible.

## Gaps in the observations

1. **The `cli.py:1696` handler body is not shown** — so we can't confirm whether it prefixes "Error:" and exits non-zero, or re-raises. This is the most important missing piece for the user-facing story.
2. **`reasons/mcp_server.py` is truncated at line 300 of 600** — the `main()` entry point isn't visible, so we can't verify how the eager `ImportError` is caught (if at all) by the console-script entry.
3. **No test files** were provided — the presence/absence of tests asserting these messages (e.g. `pytest.raises(ImportError, match="ftl-reasons\\[cluster\\]")`) is unknown.
4. **The `llm` aggregating extra** in `pyproject.toml` bundles `api`+`vertex`+`langfuse` but *not* `openai`; whether this is deliberate or a drift isn't evidenced by the observations.

## Topics to Explore

- [file] `reasons/cli.py` — Inspect the `except ImportError` at line 1696 to see how dependency errors are formatted and whether they exit non-zero.
- [file] `reasons/mcp_server.py` — Read the remaining 300 lines to see whether the eager `ImportError` at module import is caught in `main()` or bubbles up to the console script.
- [file] `pyproject.toml` — Verify the extras map (`api`, `vertex`, `openai`, `langfuse`, `llm`, `mcp`, `cluster`, `pg`) against the guard messages; note that `llm` omits `openai`.
- [general] `optional-dependency-test-coverage` — Search `tests/` for `pytest.raises(ImportError, ...)` assertions to confirm message text is regression-tested.
- [function] `reasons/llm.py:invoke_model` — Trace the full dispatch (`api:` / `vertex:` / `openai:` / CLI) to see how ImportError vs. FileNotFoundError vs. RuntimeError are distinguished.

## Beliefs

- `mcp-server-fails-eagerly` — `reasons/mcp_server.py` raises `ImportError` at import time when `mcp` is missing, while `reasons/mcp_client.py` defers the raise to `McpBridge.__init__` via `_require_mcp`.
- `optional-deps-follow-lazy-import-pattern` — Every optional dependency (`mcp`, `langchain-anthropic`, `langchain-google-vertexai`, `openai`, `sentence-transformers`+`scikit-learn`, `psycopg`, `langfuse`) is imported inside a module-level `try/except ImportError` that sets a boolean flag or `None` sentinel, so importing `reasons.*` never requires them.
- `reasons-require-function-raises-with-extras` — `_require_mcp`, `_require_cluster_deps`, `_require_psycopg`, `_invoke_api`, and `_invoke_openai` each raise `ImportError` whose message names either the pip extras (`ftl-reasons[...]`) or the raw package spec, forming the user-facing install hint.
- `langfuse-optional-is-silent` — `reasons/llm.py:_get_langfuse_handler` returns `None` when `langfuse` is absent (or `LANGFUSE_PUBLIC_KEY` is unset), so tracing is skipped without raising any error.
- `llm-distinguishes-import-from-binary-errors` — `invoke_model` raises `ImportError` for uninstalled Python API adapters but `FileNotFoundError` for missing CLI binaries (`claude`, `gemini`, `ollama`), as documented in its docstring.
