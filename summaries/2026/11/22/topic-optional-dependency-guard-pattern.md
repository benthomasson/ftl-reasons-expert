---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Compare how `cluster`, `pg`, `mcp`, `api`, `llm` extras are guarded; `pg.py` and `mcp_client.py` follow a similar `try/except ImportError` + `raise ImportError` shape, which may want a shared helper

# How the `cluster`, `pg`, `mcp`, `api`, and `llm` extras are guarded

## The extras map (from `pyproject.toml`, lines ~16–26)

| Extra | Packages |
|---|---|
| `test` | pytest, pytest-cov, pyyaml |
| `pg` | psycopg[binary]>=3.1 |
| `test-pg` | psycopg + pytest + pytest-cov |
| `cluster` | sentence-transformers>=2.2, scikit-learn>=1.2 |
| `mcp` | mcp>=1.0 |
| `api` | langchain-anthropic>=0.3 |
| `vertex` | langchain-google-vertexai>=2.0 |
| `langfuse` | langfuse>=2.0 |
| `openai` | openai>=1.0 |
| `llm` | langchain-anthropic + langchain-google-vertexai + langfuse |

The extras do **not** map 1:1 to a single guard site. `mcp` is guarded in *two* modules (`mcp_client.py`, `mcp_server.py`) with different strategies, and the four `api`/`vertex`/`langfuse`/`openai` extras (and their aggregate `llm`) are all guarded inside `llm.py` with independent flags.

## Four distinct guard shapes

### Shape A — Lazy boolean flag + `_require_*()` helper

Used by `reasons/cluster.py` and `reasons/mcp_client.py`.

`reasons/cluster.py` (lines ~11–31):
```python
try:
    from sentence_transformers import SentenceTransformer
    import numpy as np
    from sklearn.cluster import KMeans
    HAS_CLUSTER_DEPS = True
except ImportError:
    SentenceTransformer = None
    np = None
    KMeans = None
    HAS_CLUSTER_DEPS = False

def _require_cluster_deps():
    if not HAS_CLUSTER_DEPS:
        raise ImportError(
            "sentence-transformers and scikit-learn are required for --cluster mode. "
            "Install them with: pip install 'ftl-reasons[cluster]'"
        )
```
Called from `ClusterCache.__init__`, `cluster_beliefs`, and `cluster_beliefs_intra`. The imports are rebound to `None`, but the *test* is against the boolean `HAS_CLUSTER_DEPS`, not against `SentenceTransformer is None`.

`reasons/mcp_client.py` (lines ~13–27):
```python
try:
    from mcp import ClientSession, StdioServerParameters
    from mcp.client.stdio import stdio_client
    _mcp_available = True
except ImportError:
    _mcp_available = False

def _require_mcp():
    if not _mcp_available:
        raise ImportError(
            "mcp is required for MCP server support. "
            "Install it with: pip install 'mcp>=1.0'"
        )
```
Called from `McpBridge.__init__`. Note it uses the flag but does **not** rebind `ClientSession`/`StdioServerParameters`/`stdio_client` to `None`. Currently safe because `_require_mcp()` always runs first, but a latent `NameError` if accessed without the require.

### Shape B — Lazy `None`-rebind + `_require_psycopg()`

`reasons/pg.py` (lines ~14–19 and ~93–98), the shape the prompt calls out:
```python
try:
    import psycopg
    from psycopg.rows import dict_row
except ImportError:
    psycopg = None  # type: ignore[assignment]

def _require_psycopg():
    if psycopg is None:
        raise ImportError(
            "psycopg is required for PostgreSQL support. "
            "Install it with: pip install 'psycopg[binary]>=3.1'"
        )
```
Called from `PgApi.__init__` (line ~108). This is the closest to a "pure" rebind: it tests `psycopg is None`, directly. Caveat: `dict_row` is **not** assigned in the `except` clause, so `pg.dict_row` would raise `NameError` if ever touched before `_require_psycopg()` succeeds. Today no code path does — but this is exactly the inconsistency a shared helper would eliminate.

### Shape C — Lazy, per-provider flags (no `_require_*`)

`reasons/llm.py` (lines ~21–44) is different in *kind*: it holds four independent flags:

```python
try:
    from langchain_anthropic import ChatAnthropic
    _HAS_LANGCHAIN_ANTHROPIC = True
except ImportError:
    ChatAnthropic = None
    _HAS_LANGCHAIN_ANTHROPIC = False

try:
    from langchain_google_vertexai.model_garden import ChatAnthropicVertex
    _HAS_LANGCHAIN_VERTEX = True
except ImportError:
    ChatAnthropicVertex = None
    _HAS_LANGCHAIN_VERTEX = False

try:
    from langfuse.langchain import CallbackHandler as LangfuseCallbackHandler
    _HAS_LANGFUSE = True
except ImportError:
    LangfuseCallbackHandler = None
    _HAS_LANGFUSE = False

try:
    import openai as _openai_mod
    _HAS_OPENAI = True
except ImportError:
    _openai_mod = None
    _HAS_OPENAI = False
```
Two behavioral differences from shapes A/B:

- There is **no** single `_require_llm()` helper. Each flag is checked at the point of use (the actual `_invoke_api` path is beyond the first 200 lines, so the reader can't see it here — that's a gap in the observations).
- `_get_langfuse_handler()` (`llm.py:141`) is a **soft-degrade** case: if `_HAS_LANGFUSE` is false *or* `LANGFUSE_PUBLIC_KEY` is unset, it returns `None` and tracing is silently skipped. That is qualitatively different from A/B's "raise ImportError" contract — observability is optional, feature availability is not.

So `llm.py` is the odd one out: it needs A/B's "hard fail on missing provider" for `api`/`vertex`/`openai`, and something softer for `langfuse`.

### Shape D — Eager raise at module import

`reasons/mcp_server.py` (lines ~13–19):
```python
try:
    from mcp.server.fastmcp import FastMCP
except ImportError:
    raise ImportError(
        "The 'mcp' package is required for the MCP server. "
        "Install with: pip install 'ftl-reasons[mcp]'"
    )
```
This raises **at import time**, before any function is called. That's deliberate: `reasons-mcp` is a script entrypoint (`pyproject.toml`, `[project.scripts]`), and the module's entire purpose is to serve MCP — importing it without `mcp` is nonsensical. Failing at module load gives the cleanest CLI error, and there's no meaningful "sqlite-only use" of `mcp_server` to protect.

Contrast: `mcp_client.py` uses the lazy pattern because it may be imported by `ask.py` even when the user never invokes an MCP server.

### No guard at all — `api.py`

`reasons/api.py` does **not** import `psycopg` and has no `pg` guard. Instead, `_pg_dispatch` (lines ~195–199):
```python
def _pg_dispatch(pg_conninfo, domain_id, method_name, **kwargs):
    from .pg import PgApi
    with PgApi(pg_conninfo, domain_id) as pg:
        return getattr(pg, method_name)(**kwargs)
```
The import of `PgApi` is safe even without `psycopg` because `pg.py` tolerates the missing module (Shape B); the `ImportError` surfaces from `_require_psycopg()` inside `PgApi.__init__`. So `api.py` inherits the guard from `pg.py` — SQLite-only users never touch it.

## What a shared helper would look like

`pg.py`, `cluster.py`, and `mcp_client.py` all repeat the same ~15-line body:
1. `try: import <pkg>` (sometimes several).
2. On `ImportError`: rebind names to `None` and/or set a flag.
3. Define `_require_<thing>()` that raises an `ImportError` with an extra-install hint.
4. Call `_require_thing()` at the top of every public entry point.

A `reasons/_deps.py` could collapse this:

```python
# reasons/_deps.py
def try_import(*module_names):
    """Import names, return (namespace_dict, ok). ok is False if any failed."""
    ...

def require(ok, *, feature, extra, hint=""):
    if not ok:
        raise ImportError(
            f"{feature} requires the '{extra}' extra. "
            f"Install: pip install 'ftl-reasons[{extra}]'"
        )
```

Then:
```python
# cluster.py
_deps, _HAVE = try_import("sentence_transformers", "numpy", "sklearn.cluster")
...
def cluster_beliefs(...):
    require(_HAVE, feature="--cluster mode", extra="cluster")
```

Benefits: one ImportError message template; guaranteed consistency in what happens to the missing names; no more "`dict_row` was left unbound" bugs; and a single obvious place to extend to the eager vs. lazy distinction (see below).

## Design questions the shared helper should force you to answer

1. **Eager vs. lazy**: `mcp_server.py` wants eager failure; `mcp_client.py` wants lazy failure. Don't collapse these — a helper should expose both, not force one. Consider a `require_at_import(extra)` shim for the server entrypoint.
2. **Hard vs. soft**: `langfuse` in `llm.py` should **not** raise — it's an observability nicety. A shared helper needs a `required=False` style escape hatch (or `llm.py` should keep its ad-hoc flags for langfuse). The `_get_langfuse_handler` soft-degrade is intentional and should stay that way.
3. **Hint specificity**: `cluster.py` mentions `--cluster mode` in addition to the install line; `pg.py` and `mcp_client.py` don't mention the feature, only the package. Standardizing the message template will force a decision about whether the "feature name" is a required part of the helper signature.
4. **What to rebind**: `pg.py` sets `psycopg = None`, `mcp_client.py` sets nothing, `cluster.py` sets all three names. A helper that returns the imported namespace (or `{}` on failure) removes the ambiguity at the cost of `from deps import` *; the explicit `def _require_x` wrappers may still be worth keeping for IDE friendliness.

## Observations that are missing / suspect

- `all_reasons_importerror_guards.matches` is `[]` with `total_count: 0`, yet the code clearly contains `try/except ImportError` guards. The grep pattern (`ImportError|ModuleNotFoundError|optional import|missing extra`) should have matched. This is almost certainly a data-collection bug in the observation pipeline — treat the explicit `*_guard` entries as authoritative and ignore the `all_*` aggregation.
- `llm.py`'s `_invoke_api` / provider-dispatch body (beyond line 200) is where the *actual* per-provider failure happens. Without those lines we can only infer the error semantics from `resolve_model_cmd` (which raises `ValueError`, not `ImportError`, for unknown models). If you're going to standardize `llm.py` around a helper, fetch that portion first.

## Topics to Explore

- [file] `reasons/llm.py` — The `_invoke_api` path (beyond line 200) is where `_HAS_LANGCHAIN_ANTHROPIC`, `_HAS_LANGCHAIN_VERTEX`, and `_HAS_OPENAI` are actually consumed; you need it to decide whether `llm.py` fits the shared-helper model or needs to stay bespoke.
- [function] `reasons/llm.py:_get_langfuse_handler` — The only guard in the codebase that soft-degrades instead of raising; the shared helper must preserve this behavior.
- [file] `reasons/api.py` — `_pg_dispatch` (line ~195) is the reason `api.py` needs no `pg` guard; understand it before refactoring `pg.py`.
- [general] `eager-vs-lazy-import-guard-policy` — `mcp_server.py` vs `mcp_client.py` is the clearest example of when to fail at import vs. at call; codify the rule before extracting a helper.
- [general] `optional-extra-error-message-template` — The three existing messages (`cluster`, `psycopg`, `mcp`) hint inconsistently (`--cluster mode`, "PostgreSQL support", "MCP server support"); a shared helper should standardize the template.

## Beliefs

- `pg-guard-leaves-dict-row-unbound` — In `reasons/pg.py`, the `except ImportError` clause rebinds only `psycopg = None`; `dict_row` is never assigned, so `pg.dict_row` would raise `NameError` if accessed before `_require_psycopg()` succeeds.
- `mcp-server-guard-is-eager` — `reasons/mcp_server.py` raises `ImportError` at module import time, while `reasons/mcp_client.py` defers the same condition to `_require_mcp()` inside `McpBridge.__init__`.
- `cluster-guard-tests-boolean-not-None` — `reasons/cluster.py` gates on the derived boolean `HAS_CLUSTER_DEPS`, not on `SentenceTransformer is None`; the `None` rebinds are for the call sites, not the guard.
- `langfuse-extra-degrades-silently` — `reasons/llm.py:_get_langfuse_handler` returns `None` (rather than raising) when `langfuse` is missing or `LANGFUSE_PUBLIC_KEY` is unset, making the `langfuse` extra an optional enhancement rather than a hard requirement.
- `api-defers-pg-import-to-pgapi` — `reasons/api.py` never imports `psycopg` directly; `_pg_dispatch` imports `PgApi` lazily and relies on `_require_psycopg()` in `pg.py` to surface the missing-dependency error.
- `extras-do-not-map-1to1-to-guard-modules` — The `mcp` extra is guarded by two modules with different strategies, and the `llm` extra aggregates four provider deps (`api`, `vertex`, `langfuse`, `openai`) that are each guarded in `llm.py` by independent `_HAS_*` flags.
