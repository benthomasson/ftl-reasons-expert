---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Study how this module defers imports and degrades gracefully when optional integrations are absent.

## How optional integrations are deferred and how the package degrades without them

### The two-tier dependency contract

`pyproject.toml` makes this explicit at the packaging level:

- `dependencies = []` — zero required runtime dependencies.
- Named extras: `pg`, `cluster`, `mcp`, `api`, `vertex`, `langfuse`, `openai`, `llm` (a bundle), plus `test` / `test-pg`.
- `[project.scripts]` exposes `reasons` (CLI) and `reasons-mcp` (MCP server).

That packaging choice forces the source to answer one question per optional feature: *what happens if the backing package is missing?* The code answers with three distinct strategies.

---

### Strategy 1 — Guarded module-level import with capability flags (`reasons/llm.py`)

`reasons/llm.py` is the canonical example. Lines 19–46 contain four `try/except ImportError` blocks, each with the same shape:

```python
try:
    from langchain_anthropic import ChatAnthropic
    _HAS_LANGCHAIN_ANTHROPIC = True
except ImportError:
    ChatAnthropic = None
    _HAS_LANGCHAIN_ANTHROPIC = False
```

Same treatment for `langchain_google_vertexai.model_garden.ChatAnthropicVertex` (lines 27–32), `langfuse.langchain.CallbackHandler` (lines 34–39), and `openai` (lines 41–46).

Key design details:

1. **The module still imports cleanly.** Even with zero extras installed, `import reasons.llm` succeeds. This matters because `reasons/cli.py` and `reasons/api.py` all need `llm.invoke_model` regardless of which backends are installed.
2. **The imported name keeps a value.** On failure the imported symbol is set to `None` — not deleted. That prevents `NameError` from leaking out of a code path that forgot a `_HAS_` check; it fails as `AttributeError`/`TypeError` or, more likely, via the explicit guard below.
3. **The flag is the source of truth.** Every call site checks the boolean, not the imported symbol: `if not _HAS_LANGCHAIN_ANTHROPIC:` (line 207), `if not _HAS_LANGCHAIN_VERTEX:` (line 214), `if not _HAS_LANGFUSE:` (line 162), `if not _HAS_OPENAI:` (line 251).

**Important nuance:** these are *not* deferred imports. `try/except ImportError` at module top level still attempts the import eagerly on every load; it just tolerates the failure. The "deferral" is of the *requirement*, not the import call. Actual deferred imports live in function bodies (see Strategy 4).

---

### Strategy 2 — Silent degradation for tracing (`_get_langfuse_handler`, `llm.py:160`)

Langfuse is treated differently from the LLM providers: it is *observability*, not functionality. `_get_langfuse_handler` degrades to `None`, so the caller just skips attaching a callback:

```python
def _get_langfuse_handler():
    global _langfuse_handler, _langfuse_checked
    if _langfuse_checked:
        return _langfuse_handler
    _langfuse_checked = True
    if not _HAS_LANGFUSE:
        return None
    if not os.environ.get("LANGFUSE_PUBLIC_KEY"):
        return None
    _langfuse_handler = LangfuseCallbackHandler()
    return _langfuse_handler
```

Three degradation behaviors stacked in one function:

- Package missing → `_HAS_LANGFUSE` false → `None`.
- Package present but `LANGFUSE_PUBLIC_KEY` unset → `None` (no auth errors bubble up mid-call).
- The `_langfuse_checked` flag memoizes the *negative* result too, so a missing key isn't re-tested on every LLM call.

`_invoke_api` then does `handler = _get_langfuse_handler(); if handler: config["callbacks"] = [handler]` — i.e. tracing is entirely opt-in and its absence never changes control flow.

---

### Strategy 3 — Fail-fast at the point of use, with an install hint (`llm.py:205–255`)

Provider backends are *not* silently degraded — asking for `api:claude-sonnet` without the extra is a user error, and the module says so, but only when the user actually asks:

```python
if not _HAS_LANGCHAIN_ANTHROPIC:
    raise ImportError(
        "langchain-anthropic is required for api: models. "
        "Install with: pip install 'ftl-reasons[api]'"
    )
```

Repeated verbatim for `vertex:` (→ `[vertex]`) and `openai:` (→ `[openai]`). This gives three properties worth pointing out:

- **The error names the fix**, using the exact extra from `pyproject.toml`.
- **The error type is `ImportError`**, matching the docstring contract on `invoke_model` ("Raises ImportError if API dependencies are not installed").
- **CLI models never pay this cost.** `resolve_model_cmd` and the `subprocess` path never touch the optional language-specific modules, so `reasons` with the default `claude` model works with an empty dependency set.

---

### Strategy 4 — True function-local imports

Two genuine deferred imports live inside function bodies in `llm.py`:

- `get_cost_summary` does `import copy` inside the lock body (deferring a stdlib import that is only needed when stats are requested).
- `invoke_model` does `import time` at the top of the body (only needed on the hot path, not at module load).

**Observation inconsistency worth flagging:** the reported `locate_deferred_imports` scan (pattern `^\s+(import|from) `) returned **0 matches total**, yet the pasted content of `llm.py` clearly shows `    import copy` and `    import time` indented inside functions. That scan result is a false negative — either the scan didn't cover `llm.py`, or the regex anchor was applied in a way that missed these lines. Do not rely on "0 deferred imports" as a fact about this repo.

---

### Strategy 5 — Hard fail at import time when the module is meaningless without the dependency (`reasons/mcp_server.py:15–19`)

```python
try:
    from mcp.server.fastmcp import FastMCP
except ImportError:
    raise ImportError(
        "The 'mcp' package is required for the MCP server. "
        "Install with: pip install 'ftl-reasons[mcp]'"
    )
```

`mcp_server.py` uses the *opposite* policy from `llm.py`: there is no `_HAS_MCP` flag, no `None` sentinel — the module re-raises with a friendly message. That's deliberate: `reasons-mcp` is an entry point in `pyproject.toml`, so if `mcp` isn't installed, running `reasons-mcp` will never do anything useful. Failing on import gives a clear error instead of exposing a batteries-not-included server object.

Contrast this with `llm.py`: `reasons` (the main CLI) must always work, so `llm` tolerates missing API extras. `reasons-mcp` has no fallback mode, so it fails loud.

---

### Strategy 6 — Same spirit, other modules (inferred from the match list)

The observations show `except ImportError:` at:

- `reasons/cluster.py:17` — almost certainly guarding `sentence-transformers` / `scikit-learn` from the `[cluster]` extra.
- `reasons/pg.py:18` — almost certainly guarding `psycopg[binary]` from the `[pg]` extra.
- `reasons/mcp_client.py:17` — almost certainly guarding `mcp` for client use.
- `reasons/cli.py:1696` — `except ImportError as e:`, likely inside a subcommand handler that surfaces the message to the user rather than crashing argparse.

The **contents of these blocks are not included** in the observations, so we can't confirm whether they follow the `_HAS_FLAG` + `None` pattern from `llm.py` or the hard-fail pattern from `mcp_server.py`. Given the extras in `pyproject.toml` and the fact each module has its own `except`, the likely shape is `_HAS_*` + `None` for `cluster.py`/`pg.py`/`mcp_client.py` and message-wrapping in `cli.py`. **This should be verified before asserting it.**

---

### Strategy 7 — Avoid the dependency entirely (`reasons/hf.py`)

`reasons/hf.py` is a third pattern: instead of `try/except ImportError` around `huggingface_hub`, the module talks to `https://huggingface.co` over `urllib.request` using only stdlib (`json`, `os`, `pathlib`, `urllib.error`, `urllib.request`). Token resolution reads `HF_TOKEN` or `~/.cache/huggingface/token` (`_resolve_token`). The cost of not depending on `huggingface_hub` is a hand-rolled `create_repo` (handling HTTP 409 as "already exists") and `upload_file` (PUT to `/api/models/.../upload/...`), but the benefit is that HF import/upload ships with the base install. This is a deliberate "no optional dependency at all" choice — the strongest form of graceful degradation.

---

### Non-import graceful degradation in the same style (`reasons/check_stale.py`)

`check_stale.py` shows the same philosophy applied to runtime failure rather than import failure. Lines 222 and 273 note that git errors return `True` to *force fallback to content hashing*. Same discipline: an optional integration (the `git` binary / repo working tree) is present enough to try, its absence is detected at runtime, and the caller falls back to a strictly-weaker path rather than failing.

---

### Test coverage gap

`locate_tests_for_missing_optional_deps` (pattern `ImportError|ModuleNotFoundError|sys.modules|monkeypatch`) returned **0 matches**, yet the earlier guard scan found `except ImportError` in `tests/test_api.py:783`, `tests/test_cli.py:817`, `tests/test_cluster.py:14`, and `tests/test_contradictions.py:377`. As with the deferred-import scan, the second scan appears to be a false negative. What we can say from the surviving matches is:

- Those test files use `except ImportError` (the common `try: import pytest_foo except ImportError: pytest.skip(...)` idiom), which **skips** tests when an extra is missing rather than exercising the degradation path.
- No `monkeypatch`-based tests were positively identified that simulate `ImportError` inside `reasons.llm` to exercise `_invoke_api`'s guidance message or `_get_langfuse_handler`'s `None` path. That's a real coverage hole: the graceful-degradation branches are only exercised in the environment they were designed for (deps installed), not in the environment they're designed to survive (deps missing).

---

## Summary table

| Module | Feature | Pattern | Behavior when dep is missing |
|---|---|---|---|
| `reasons/llm.py` | `langchain_anthropic` | `_HAS_FLAG` + `None` | import succeeds; `api:` raises `ImportError` with `[api]` hint |
| `reasons/llm.py` | `langchain_google_vertexai` | `_HAS_FLAG` + `None` | import succeeds; `vertex:` raises `ImportError` with `[vertex]` hint |
| `reasons/llm.py` | `langfuse` | `_HAS_FLAG` + memoized getter | tracing silently disabled (`None`) |
| `reasons/llm.py` | `openai` | `_HAS_FLAG` + `None` | import succeeds; `openai:` raises `ImportError` with `[openai]` hint |
| `reasons/mcp_server.py` | `mcp` | hard-fail at import | raises `ImportError` with `[mcp]` hint immediately |
| `reasons/hf.py` | HuggingFace Hub | no dependency | always available (stdlib `urllib`) |
| `reasons/cluster.py:17` | clustering backend | `except ImportError:` (body unknown) | not determinable from observations |
| `reasons/pg.py:18` | Postgres backend | `except ImportError:` (body unknown) | not determinable from observations |
| `reasons/mcp_client.py:17` | `mcp` client | `except ImportError:` (body unknown) | not determinable from observations |
| `reasons/cli.py:1696` | subcommand | `except ImportError as e:` | wraps the error for CLI display (likely) |
| `reasons/check_stale.py:222,273` | `git` binary | runtime try/except | falls back to content hashing |

---

## Topics to Explore

- [file] `reasons/cluster.py` — Verify whether the `except ImportError` at line 17 uses the `_HAS_*` flag pattern or the hard-fail pattern, and which specific symbols are guarded.
- [file] `reasons/pg.py` — Confirm how `psycopg` absence is represented (module-level `None`, capability flag, or deferred connection error) since it gates the entire `[pg]` extra.
- [function] `reasons/llm.py:_invoke_api` — Read the full body to see whether `resolve_model_cmd`'s openai/vertex rejection messages are consistent with the `ImportError` guidance raised here.
- [file] `reasons/mcp_client.py` — Compare with `reasons/mcp_server.py`: does the client degrade silently while the server hard-fails, or do both fail at import?
- [general] `optional-dependency-test-strategy` — Search the test tree for `monkeypatch` / `sys.modules[...] = None` patterns to confirm whether any test actually asserts behavior when an extra is *not* installed.

## Beliefs

- `module-load-does-not-require-optional-deps` — `import reasons.llm` succeeds with no extras installed because each optional import (lines 19–46) is wrapped in `try/except ImportError` and assigns `None` to the symbol on failure.
- `capability-flag-is-source-of-truth` — Every consumer of an optional symbol in `reasons/llm.py` checks the `_HAS_*` boolean (lines 162, 207, 214, 251) rather than testing the imported symbol for `None`, so the flag and the sentinel are kept in sync at load time.
- `api-models-raise-import-error-at-call-time` — `_invoke_api` and `_invoke_openai` raise `ImportError` only when an `api:`/`vertex:`/`openai:` model is requested, and the message names the corresponding `pip install 'ftl-reasons[<extra>]'` command.
- `langfuse-missing-is-silent` — `_get_langfuse_handler` returns `None` when either `_HAS_LANGFUSE` is false or `LANGFUSE_PUBLIC_KEY` is unset, and memoizes that result via `_langfuse_checked` so the negative path is not retested per call.
- `mcp-server-fails-at-import` — `reasons/mcp_server.py` re-raises `ImportError` at module load (line 17), unlike `reasons/llm.py`, because `reasons-mcp` has no useful degraded mode.
- `hf-integration-uses-stdlib-only` — `reasons/hf.py` imports only `json`, `os`, `pathlib`, and `urllib`, so the `[hf]`-style extra does not exist and the module always works with a bare install.
- `deferred-import-scan-is-false-negative` — The `locate_deferred_imports` result of 0 total matches contradicts the pasted `llm.py` content, which contains indented `import copy` inside `get_cost_summary` and `import time` inside `invoke_model`; the scan result should not be trusted as evidence of zero deferred imports.
