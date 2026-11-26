---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: How the rest of `reasons` gates optional heavy imports (e.g. `sentence-transformers`, `huggingface_hub`) and whether the same ImportError-to-exit-1 idiom is used consistently.

# How `reasons` gates optional heavy imports — and whether ImportError→exit-1 is consistent

## The gating pattern

The codebase uses a single, repeated pattern for optional heavy imports:

1. **Module-level guarded import** — a `try/except ImportError` binds the symbol to `None` on failure and records a boolean capability flag (e.g. `HAS_*` / `_HAS_*`). The module therefore imports successfully even when the dependency is absent.
2. **Call-time guard** — the actual heavy path calls a helper that checks the flag and raises `ImportError` with an actionable pip hint, instead of failing at import.
3. **Packaging** — the pip extras named in those hints are declared in `pyproject.toml` (`[project.optional-dependencies]`, line 14; `cluster = ["sentence-transformers>=2.2", "scikit-learn>=1.2"]`, line 18).

### `reasons/cluster.py` — `sentence-transformers` / `scikit-learn`

The guarded import is at the top of the module (~lines 11–20):

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
```

At call time, every entry point (`ClusterCache.__init__`, `cluster_beliefs`, `cluster_beliefs_intra`, `list_clusters`) calls `_require_cluster_deps()` first:

```python
def _require_cluster_deps():
    if not HAS_CLUSTER_DEPS:
        raise ImportError(
            "sentence-transformers and scikit-learn are required for --cluster mode. "
            "Install them with: pip install 'ftl-reasons[cluster]'"
        )
```

So the module **raises** `ImportError` — it does not call `sys.exit(1)` itself.

### `reasons/llm.py` — `langchain-anthropic`, `vertexai`, `langfuse`, `openai`

The same shape, four times (`llm.py` ~lines 16–42):

- `langchain_anthropic` → `_HAS_LANGCHAIN_ANTHROPIC`
- `langchain_google_vertexai.model_garden.ChatAnthropicVertex` → `_HAS_LANGCHAIN_VERTEX`
- `langfuse.langchain.CallbackHandler` → `_HAS_LANGFUSE`
- `openai` → `_HAS_OPENAI`

The call-time guards are spread across `_invoke_api` and `_invoke_openai`:

- `api:` → `ImportError("... pip install 'ftl-reasons[api]'")`
- `vertex:` → `ImportError("... pip install 'ftl-reasons[vertex]'")`
- `openai:` → `ImportError("... pip install 'ftl-reasons[openai]'")`

`_get_langfuse_handler()` is deliberately *soft*: if `_HAS_LANGFUSE` is false it simply returns `None` and tracing is skipped, rather than raising.

### `reasons/hf.py` — no `huggingface_hub` at all

`hf.py` is the odd one out, and this is the important nuance: **it never imports `huggingface_hub`**. It implements HuggingFace access directly on the standard library:

```python
from urllib.error import HTTPError
from urllib.request import Request, urlopen
```

`download_network`, `create_repo`, and `upload_file` all use `urlopen(...)` against `https://huggingface.co/...` and translate `HTTPError` (401/404/409) into `RuntimeError`. So there is no heavy import to gate — the dependency was designed away, not guarded.

## Is the same ImportError→exit-1 idiom used consistently?

**No — and arguably the ImportError→exit-1 idiom doesn't exist as such in the shown code.**

Two distinct conventions are in play:

- **Library modules (`cluster.py`, `llm.py`)** raise `ImportError` with a pip-install hint. They do **not** call `sys.exit(1)`.
- **CLI (`cli.py`)** owns the exit-1 ceremony, but only for *domain* errors. Every `cmd_*` wrapper does `except (ValueError | KeyError | IndexError | FileExistsError) as e: print(..., file=sys.stderr); sys.exit(1)` (e.g. `cmd_init`, `cmd_add`, `cmd_add_justification`, `cmd_retract`, `cmd_mark_duplicate`). There is **no `except ImportError:` in the 400 lines of `cli.py` shown** — so an `ImportError` raised by `cluster.py`/`llm.py` is not visibly converted to exit 1; it would propagate (likely as a traceback) unless something upstream catches it.

So the consistent idiom is "**catch → print to stderr → `sys.exit(1)`**" for domain exceptions, while optional-dependency failures use "**raise `ImportError` with an install hint**" and rely on an outer layer (not shown) to present it.

## Reliability caveat on the automated searches

The "locate" passes are **contradicted by the file contents** and should not be trusted for consistency claims:

- `locate_optional_heavy_imports` (pattern `sentence[_-]?transformers|huggingface[_-]?hub`) returned **0 matches**, yet `reasons/cluster.py` clearly contains `from sentence_transformers import SentenceTransformer`.
- `locate_importerror_handling` and `locate_exit1_idiom` also returned **0**, despite obvious `except ImportError:` and `sys.exit(1)` in the shown files.

This indicates the regex/grep layer is truncated or mis-scoped; the search results cannot be used as evidence either way. (Note also only `cli.py` lines 1–400 of 4015 and `llm.py` lines 1–300 of 327 were inspected.)

## What's missing to fully answer "consistently"

- **`reasons/api.py` was not inspected** — the most likely place the `ImportError` from `cluster.py`/`llm.py` is translated into a user-facing message.
- **Lines 401–4015 of `cli.py`** — the remaining ~90% could contain a top-level `except ImportError` or a dispatch wrapper.
- **The rest of `pyproject.toml`** — only the `cluster` extra is visible; the `api`, `vertex`, `openai` extras referenced by `llm.py` hints aren't shown.
- **`llm.py` lines 301–327** — the tail of `invoke_model` may handle the propagated `ImportError`.

## Topics to Explore

- [file] `reasons/api.py` — Where module-level `ImportError`s from `cluster.py`/`llm.py` would be caught and turned into user-facing errors
- [function] `reasons/cli.py:cmd_classify_defeaters` — An LLM-dependent CLI command; check how it surfaces `ImportError` when `[api]`/`[openai]` extras are missing
- [general] `optional-dependency-extras` — How `pyproject.toml` declares the `api`/`vertex`/`openai` extras that `llm.py` hints reference
- [function] `reasons/cluster.py:_require_cluster_deps` — Canonical reference implementation of the raise-ImportError-with-hint pattern
- [file] `reasons/llm.py` — Remaining lines after 300, to see if `invoke_model` translates `ImportError` at its boundary

## Beliefs

- `heavy-imports-gated-by-module-level-try-except` — Optional heavy imports are wrapped in module-level `try/except ImportError` that set `*_HAS_*`/`HAS_*` flags and `None` sentinels instead of failing import.
- `cluster-mode-requires-sentence-transformers-extra` — `reasons/cluster.py` requires the `cluster` extra (`sentence-transformers` + `scikit-learn`), enforced at call time by `_require_cluster_deps()`.
- `hf-module-avoids-huggingface-hub` — `reasons/hf.py` implements HuggingFace HTTP via stdlib `urllib.request` and does not import `huggingface_hub`, so no gating exists for it.
- `llm-module-raises-importerror-with-extra-hints` — `reasons/llm.py`'s `_invoke_api` and `_invoke_openai` raise `ImportError` naming the pip extra (`[api]`, `[vertex]`, `[openai]`) rather than exiting.
- `cli-exception-to-exit1-idiom` — `cmd_*` functions in `reasons/cli.py` catch `ValueError`/`KeyError`/`IndexError`/`FileExistsError`, print to `stderr`, and `sys.exit(1)`; no `except ImportError` appears in the shown lines.
