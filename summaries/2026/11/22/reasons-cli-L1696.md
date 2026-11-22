---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Line 1696 in reasons/cli.py

# Line 1696 — `except ImportError as e:`

This line is the exception handler for the `try` block that conditionally imports and instantiates `ClusterCache` inside `cmd_derive`. It is the guard that converts a missing optional clustering dependency into a clean CLI error.

```python
1689  if args.cluster or args.intra_cluster:
1690      try:
1691          from .cluster import ClusterCache
1692          print("Loading embedding model...", file=sys.stderr)
1693          cluster_cache = ClusterCache(
1694              model_name=args.embedding_model or "all-MiniLM-L6-v2"
1695          )
1696      except ImportError as e:          # ← the symbol
1697          print(f"Error: {e}", file=sys.stderr)
1698          sys.exit(1)
```

## 1. Purpose

The clustering feature of `derive` is *optional*: it pulls in `sentence-transformers` (and transitively `torch`/`numpy`) only when the user passes `--cluster` or `--intra-cluster`. Line 1696 exists to make a missing dependency a graceful, actionable CLI failure instead of a raw traceback. Without it, a user running `reasons derive --cluster` on a machine without the optional embedding stack installed would see an ugly `ModuleNotFoundError` stack trace; with it, they see `Error: No module named 'sentence_transformers'` and exit code 1.

## 2. Contract

- **Precondition**: Reached only when `args.cluster or args.intra_cluster` is truthy (line 1689). If neither flag is set, execution never enters the `try` and this line is never executed.
- **Postcondition (on `ImportError`)**: The exception's string form is written verbatim to `stderr`, the process exits with status `1`. No `cluster_cache` is created; the caller (`cmd_derive`) never sees the exception because `SystemExit` propagates out (which argparse/`main` treat as normal termination).
- **Invariant**: Only `ImportError` (including `ModuleNotFoundError`) is caught. All other exceptions from `ClusterCache(...)` propagate.
- **Assumption not enforced by types**: The block assumes that any `ImportError` in this region means "optional dependency is missing." That is not guaranteed. An internal typo in `.cluster`'s own imports, or a circular-import bug, would also surface as `ImportError` and be misreported to the user as a missing package.

## 3. Parameters

- `e: ImportError` — The caught exception. Python populates `.name` (missing module) and `.path` (search path) on it. The code interpolates `str(e)` directly, so the printed text is exactly what Python would have said in the traceback (e.g. `No module named 'sentence_transformers'`). The code does not use `.name` or `.path` explicitly — so consumers only get the generic message, not a suggested `pip install`.

## 4. Return Value

None. This line always calls `sys.exit(1)` — it terminates the process rather than returning to the caller. `cmd_derive` will not resume from this path; any following code (report writing, cost summary) is skipped.

## 5. Algorithm (context of the block)

1. Enter `cmd_derive`. After parsing `--prompt-file` and validating mutually-exclusive flag combinations (`--cluster`/`--intra-cluster`/`--sample`), execution reaches line 1688.
2. `if args.cluster or args.intra_cluster:` — the clustering path is only taken when the user opted in.
3. Enter the `try`. Two things can raise `ImportError`:
   - `from .cluster import ClusterCache` — fails if `.cluster` doesn't exist, or if its own top-level imports (e.g. `sentence_transformers`) are missing.
   - `ClusterCache(model_name=...)` — if the constructor lazily imports `SentenceTransformer` or `torch`, that import can fail here too. (The comment `Loading embedding model...` on line 1692 is printed **before** this constructor call, so if the constructor raises `ImportError`, the user sees both the loading message *and* the error — misleading, since the model was never actually loaded.)
4. On `ImportError`, line 1696 catches it, line 1697 writes `Error: <message>` to `stderr`, and line 1698 exits.
5. If no `ImportError` occurs, `cluster_cache` is populated and flows into `_derive_one_round(..., cluster_cache=cluster_cache, ...)` at the bottom of `cmd_derive`.

## 6. Side Effects

- **Writes to `stderr`** via `print(..., file=sys.stderr)` — the user sees the error message but the pipe on stdout is left untouched.
- **Calls `sys.exit(1)`** — raises `SystemExit(1)`, terminating the process. Any `finally` blocks higher up (e.g. `cost = format_cost_summary()`) do **not** run on this path because `SystemExit` is raised inside `cmd_derive` before those statements.
- **No partial state is created** (no cache object is left half-initialized because the assignment `cluster_cache = ...` only completes if the constructor returns).

## 7. Error Handling

- **Caught**: `ImportError` — covers `ModuleNotFoundError` and any import-time failure during `.cluster` load or `ClusterCache` init.
- **Not caught** (will crash the CLI with a traceback):
  - `RuntimeError` / `OSError` — e.g. sentence-transformers tries to download `all-MiniLM-L6-v2` and the network is down, or the HF cache is corrupt.
  - `ValueError` — e.g. invalid model name passed via `--embedding-model`.
  - `KeyError` / `AttributeError` — bugs in `ClusterCache.__init__`.
- **Silent hazard**: The code cannot distinguish "package not installed" from "package is installed but broken", so a real bug inside `.cluster` may be presented to the user as "No module named X" and lead to wasted `pip install` debugging.

## 8. Usage Patterns

- Triggered only from `reasons derive --cluster` or `reasons derive --intra-cluster` (and only when combined with the other round-loop flags at the bottom of `cmd_derive`).
- Callers (i.e. end users of the CLI) are expected to install the optional clustering extras first — e.g. `pip install reasons[cluster]` or `pip install sentence-transformers`.
- This mirrors the error style used elsewhere in `cli.py` (e.g. `cmd_init` catching `FileExistsError`, `cmd_review_beliefs` catching generic `Exception`) — print to stderr, `sys.exit(1)`.
- Because `cluster_cache` may be `None` (when both flags are absent), any downstream code consuming it (`build_prompt` in `_derive_one_round`) must handle `cluster_cache=None` as "no clustering" — that contract lives in `.derive`, not here.

## 9. Dependencies

- `sys` (module-level import in `cli.py`).
- `from .cluster import ClusterCache` — deferred import (line 1691) so that users who never enable clustering do not pay the import cost of `sentence-transformers`/`torch`.
- `args.embedding_model`, `args.cluster`, `args.intra_cluster` — argparse-derived attributes registered in `main()`'s `derive` subparser (`--cluster`, `--intra-cluster`, `--embedding-model`).
- `ClusterCache` itself — its constructor signature must accept `model_name: str` (relied on, not type-checked at this call site).
- Downstream: `_derive_one_round` consumes `cluster_cache` and passes it to `build_prompt` in `.derive`.

## Topics to Explore

- [file] `reasons/cluster.py` — The `ClusterCache` class whose import and constructor sit inside this `try`; check what it actually imports and whether `ImportError` can come from non-dependency sources.
- [function] `reasons/derive.py:build_prompt` — Consumes `cluster_cache` and `embedding_model`; verify how `None` is treated and what `all-MiniLM-L6-v2` defaulting means there.
- [function] `reasons/cli.py:main` — Where `--cluster`, `--intra-cluster`, and `--embedding-model` are registered; confirm defaults are `False`, `False`, `None`.
- [general] `optional-dependencies-pattern` — How the rest of `reasons` gates optional heavy imports (e.g. `sentence-transformers`, `huggingface_hub`) and whether the same ImportError-to-exit-1 idiom is used consistently.
- [general] `exhaust-round-loop` — How `cluster_cache` is reused across `--exhaust` rounds and whether the "Loading embedding model..." message is printed once or per-call.

## Beliefs

- `cluster-flags-gate-clustercache-import` — `ClusterCache` is imported only when `args.cluster` or `args.intra_cluster` is truthy; the import is otherwise skipped to avoid the `sentence-transformers`/`torch` cost.
- `clustercache-importerror-produces-exit-code-1` — Any `ImportError` raised by the `.cluster` import or by `ClusterCache.__init__` is caught and converted into a `stderr` message plus `sys.exit(1)`.
- `clustercache-non-importerror-exceptions-propagate` — `RuntimeError`, `OSError`, `ValueError`, and other non-`ImportError` exceptions from `ClusterCache(...)` are not caught by this handler and will crash `derive` with a traceback.
- `loading-message-printed-before-model-load` — The `Loading embedding model...` message (line 1692) is emitted before `ClusterCache(...)` is constructed, so it is visible even when the constructor later fails with an `ImportError`.
- `default-embedding-model-is-all-minilm-l6-v2` — When `--embedding-model` is not supplied, `ClusterCache` is instantiated with `all-MiniLM-L6-v2` via `args.embedding_model or "all-MiniLM-L6-v2"`.
