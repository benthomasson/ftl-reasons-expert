---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: The three existing messages (`cluster`, `psycopg`, `mcp`) hint inconsistently (`--cluster mode`, "PostgreSQL support", "MCP server support"); a shared helper should standardize the template.

# Standardizing the optional-dependency hint template

## What the observations show

Three modules each embed a "you need to install something" message, but they are textually different in four dimensions: **preamble**, **verb**, **package spec**, and **feature name**.

| File | Line | Text |
|---|---|---|
| `reasons/cluster.py` | 30 | `"Install them with: pip install 'ftl-reasons[cluster]'"` |
| `reasons/pg.py` | 93–94 | `"psycopg is required for PostgreSQL support. "` + `"Install it with: pip install 'psycopg[binary]>=3.1'"` |
| `reasons/mcp_client.py` | 24–25 | `"mcp is required for MCP server support. "` + `"Install it with: pip install 'mcp>=1.0'"` |
| `reasons/mcp_server.py` | 20 | `"Install with: pip install 'ftl-reasons[mcp]'"` |
| `reasons/llm.py` | 210, 217, 254 | `"Install with: pip install 'ftl-reasons[api]'"` / `[vertex]` / `[openai]` |

### The four axes of drift

1. **Preamble.** `pg.py:93` and `mcp_client.py:24` open with `"<module> is required for <feature>. "`. `cluster.py:30`, `mcp_server.py:20`, and `llm.py:210/217/254` have no preamble at all — the reader sees only the install command with no explanation of *why*.

2. **Install verb.** Three distinct forms exist:
   - `"Install it with:"` — `pg.py:94`, `mcp_client.py:25`
   - `"Install them with:"` — `cluster.py:30` (plural "them" has no antecedent, since the preceding runtime message at `cluster.py:30` doesn't list multiple modules)
   - `"Install with:"` — `mcp_server.py:20`, `llm.py:210`, `llm.py:217`, `llm.py:254`

3. **Package spec — the real bug.** Two modules point at **bare PyPI packages**, three point at the **project extras**:
   - `reasons/mcp_server.py:20` tells the user to `pip install 'ftl-reasons[mcp]'`, but `reasons/mcp_client.py:25` tells them to `pip install 'mcp>=1.0'` — same extra, two different commands. `pyproject.toml` defines `mcp = ["mcp>=1.0"]`, so `ftl-reasons[mcp]` is the canonical install path and `mcp>=1.0` is the fallback. Only one of these should be advertised.
   - Same split for Postgres: `pyproject.toml` defines `pg = ["psycopg[binary]>=3.1"]`, yet `pg.py:94` advertises the bare wheel instead of `ftl-reasons[pg]`.
   - `cluster.py` (`ftl-reasons[cluster]`) and `llm.py` (`ftl-reasons[api]`/`[vertex]`/`[openai]`) already use the extra form.

4. **Feature name.** `"PostgreSQL support"`, `"MCP server support"`, and the docstring at `cluster.py:6` (`"Requires: pip install 'ftl-reasons[cluster]'"`) present three different shapes. Notably:

   > `find_cluster_message` searched for `--cluster mode` and found **zero matches**. The phrase "cluster mode" never appears in the code — `reasons/cli.py` uses `--pg` style flags (`getattr(args, "pg", None)`), not `--cluster`. So the intended canonical phrasing doesn't yet exist anywhere; it has to be invented by the helper.

### A latent semantic bug in `mcp_client.py`

`reasons/mcp_client.py:24` says `"mcp is required for MCP server support."` — but this is the **client** module. `reasons/mcp_server.py:20` also talks about MCP. If you standardize the message without catching this, you'll propagate a wrong feature label across both modules. The helper should take `feature` as a required argument, which forces each call site to state it correctly (and surfaces this bug during migration).

### Where the trailing space comes from

`pg.py:93` and `mcp_client.py:24` end in a trailing space. That strongly implies **implicit string concatenation** of a two-line message — either a multi-line `raise ImportError(...)` or a two-line `print(...)`. The helper should return a single propagated string, so callers never have to remember the space.

## Proposed shared helper

Put it in a small private module (e.g. `reasons/_deps.py`) so both `reasons/` and `reasons_lib/` can import it — `reasons_lib/cli.py` is only a 3-line shim (`from reasons.cli import *`), so it inherits everything.

```python
# reasons/_deps.py
from __future__ import annotations

def optional_dep_hint(
    module: str,
    feature: str,
    extra: str,
    *,
    package: str = "ftl-reasons",
) -> str:
    """Standardised message for a missing optional dependency.

    >>> optional_dep_hint("psycopg", "PostgreSQL support", "pg")
    "psycopg is required for PostgreSQL support. "
    "Install it with: pip install 'ftl-reasons[pg]'"
    """
    return (
        f"{module} is required for {feature}. "
        f"Install it with: pip install '{package}[{extra}]'"
    )
```

Design decisions worth stating to the reviewer:

- **`module` first, feature second, extra third** — mirrors the natural English *"X is required for Y"* and keeps the call site reading like a sentence.
- **Always route through `ftl-reasons[<extra>]`**, never the bare package. `pyproject.toml` already owns version pinning (`psycopg[binary]>=3.1`, `mcp>=1.0`, `sentence-transformers>=2.2` + `scikit-learn>=1.2`); duplicating those strings in source is what caused the drift in the first place.
- **No trailing space in the return value** — the helper owns the whitespace, not the caller.
- **`feature` is required, not defaulted** — deliberately forces `mcp_client.py` to be relabeled (e.g. `"MCP client support"`) rather than silently inheriting the wrong string.

### Call-site migration table

| File:line | After migration |
|---|---|
| `reasons/cluster.py:6` (docstring) | `optional_dep_hint("sentence-transformers", "clustering support", "cluster")` |
| `reasons/cluster.py:30` | same call — one source of truth for docstring + runtime |
| `reasons/pg.py:6` | same as `pg.py:94` |
| `reasons/pg.py:93-94` | `optional_dep_hint("psycopg", "PostgreSQL support", "pg")` |
| `reasons/mcp_client.py:6` | same as `mcp_client.py:25` |
| `reasons/mcp_client.py:24-25` | `optional_dep_hint("mcp", "MCP client support", "mcp")` |
| `reasons/mcp_server.py:20` | `optional_dep_hint("mcp", "MCP server support", "mcp")` |
| `reasons/llm.py:210` | `optional_dep_hint("langchain-anthropic", "API support", "api")` |
| `reasons/llm.py:217` | `optional_dep_hint("langchain-google-vertexai", "Vertex AI support", "vertex")` |
| `reasons/llm.py:254` | `optional_dep_hint("openai", "OpenAI support", "openai")` |
| `tests/test_cluster.py:4` | Build the expected string via the helper, or assert on `package`+`extra` substrings, so the test doesn't re-encode the template |

## What's missing from the observations

- **The bodies of `cluster.py`, `pg.py`, `mcp_client.py`, `mcp_server.py`, and `llm.py` are not shown** — only grep hits. We can't confirm whether the messages are delivered via `raise ImportError(...)`, `print(..., file=sys.stderr)`, or returned in a dict. The helper's return type should match whichever pattern dominates; if it's `ImportError`, a thin `raise_missing_dep(...)` wrapper may be nicer than returning a string.
- **No existing helper module is visible.** Whether `reasons/_deps.py`, `reasons/_utils.py`, or some `reasons/deps.py` already exists can't be determined from the observations.
- **The cluster docstring at `cluster.py:6` uses `'ftl-reasons[cluster]'`** while `cluster.py:30` uses the same string — but we don't see whether the docstring is generated from the same constant, so we can't say whether docstring drift is already a risk.
- **`tests/test_cluster.py:4`** presumably *asserts* on the message text. If so, migrating the string will break the test unless the assertion is rewritten against the helper's output.

## Topics to Explore

- [file] `reasons/pg.py` — See how the `ImportError` at lines 93–94 is raised (and whether `psycopg` is imported lazily inside the raise site or at module top).
- [file] `reasons/mcp_client.py` — Confirm the client-vs-server naming bug at line 24 and how the `mcp` import is guarded.
- [file] `reasons/llm.py` — The three `"Install with:"` strings at lines 210/217/254 are the cleanest existing template; replicate its shape.
- [function] `reasons/cli.py:_backend_kwargs` — Explains how backend selection flows, and confirms there is no `--cluster` flag (only `--pg`), which is why `--cluster mode` never appears.
- [general] `dependency-extra-taxonomy` — Map every extra in `pyproject.toml` (`pg`, `cluster`, `mcp`, `api`, `vertex`, `langfuse`, `openai`, `llm`) to the module that consumes it, so the helper's `extra` argument stays exhaustive.

## Beliefs

- `mcp-client-hint-misnames-feature` — `reasons/mcp_client.py:24` labels MCP as "MCP server support" even though the file is the client module.
- `pg-and-mcp-hints-bypass-extras` — `reasons/pg.py:94` and `reasons/mcp_client.py:25` recommend bare packages (`psycopg[binary]>=3.1`, `mcp>=1.0`), whereas `reasons/mcp_server.py:20` recommends `ftl-reasons[mcp]` for the same extra.
- `cluster-hint-uses-plural-install-verb` — `reasons/cluster.py:30` says `"Install them with:"` with no preceding list of modules, so the pronoun has no antecedent.
- `cluster-mode-phrase-absent` — The literal string `--cluster mode` has zero matches in the codebase; the CLI's backend flag is `--pg`.
- `optional-dep-message-is-raw-string-concatenation` — The trailing spaces on `reasons/pg.py:93` and `reasons/mcp_client.py:24` indicate the message is assembled from two adjacent string literals rather than a shared template.
- `mcp-extra-is-valid-install-command` — `pyproject.toml` defines `mcp = ["mcp>=1.0"]`, so `pip install 'ftl-reasons[mcp]'` (used by `mcp_server.py:20`) is a supported alternative to the bare `mcp` package.
