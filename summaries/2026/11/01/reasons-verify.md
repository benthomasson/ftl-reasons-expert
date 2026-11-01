---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/verify.py

# `reasons/verify.py` — Belief Source-Verification via LLM

## Purpose

This module owns the **source-grounding verification** step in the knowledge-base pipeline. Given a set of beliefs (claims) and their associated source documents, it asks an LLM to judge whether each claim is actually supported by the text. The output is a per-belief verdict: `CONFIRMED`, `STALE`, `PARTIAL`, or `INCONCLUSIVE`, accompanied by a reason and a verbatim quote.

The docstring explicitly positions this as *rigorous source-checking* as opposed to `report_belief.py`, which does narrative synthesis. The distinction matters: here, the LLM is acting as an auditor, not a writer.

## Key Components

| Component | Contract |
|---|---|
| `MAX_SOURCE_CHARS` (30 000) | Hard ceiling on source text passed to the LLM. Prevents token-bombing on large documents. |
| `VERIFY_PROMPT` | A Jinja-style `.format()` template. Expects one `{beliefs}` slot. Instructs the LLM to return a strict JSON object. |
| `read_source(source, db_path=None) -> str \| None` | Resolves a source path via `check_stale.resolve_source_path`, reads the file, truncates to `MAX_SOURCE_CHARS`. Returns `None` on any failure (missing file, I/O error, decode error). Never raises. |
| `format_belief_for_verify(belief, source_content) -> str` | Renders one belief block for the prompt: id, claim text, source file/URL, and the source content in a fenced code block. Pure string assembly, no side effects. |
| `parse_verify_response(response) -> dict` | Extracts the first `{...}` block via regex, parses JSON, validates each entry has a string `"verdict"` key. Returns `{}` on any malformed input. Normalises verdicts to uppercase. |
| `verify_beliefs(beliefs_with_sources, model, timeout) -> dict` | The public entry point. Takes a list of `(belief_dict, source_content_or_None)` tuples, assembles the full prompt, calls `llm.invoke_model`, and returns the parsed verdict map. |

### `read_source` specifics

```python
db_dir = Path(db_path).parent if db_path else None
path = resolve_source_path(source, db_dir=db_dir)
```

It derives a *directory* from `db_path` (which is a file path to the belief database) and passes it as `db_dir` to `resolve_source_path`. This means source paths are resolved *relative to the database's location*, not CWD. If `db_path` is `None`, it falls through to whatever default `resolve_source_path` uses.

Truncation appends the literal marker `"\n\n[... truncated ...]"` so the LLM knows the source was cut.

### `parse_verify_response` specifics

The regex `r"\{.*\}"` with `re.DOTALL` grabs the first brace-delimited block. This is deliberately permissive—it handles LLMs that wrap JSON in markdown fences or prepend explanatory text. The subsequent per-entry validation (`"verdict" not in v`, `isinstance(raw_verdict, str)`) silently drops malformed entries rather than failing the whole batch.

## Patterns

- **Lazy / deferred imports.** Both `check_stale.resolve_source_path` and `llm.invoke_model` are imported *inside* the functions that use them, not at module top-level. This avoids circular-import risk (the `reasons` package has dense cross-references) and defers the cost of importing `llm` (which may pull in API-key configuration) until it's actually needed.

- **Graceful degradation over exceptions.** Every fallible path in this module returns a "safe empty" value (`None`, `{}`). The caller in `api.py` is responsible for interpreting "no verdict" as "verification was inconclusive." There is no exception class, no retry logic, no logging—this is a deliberately thin, side-effect-light module.

- **Template + parser separation.** The prompt (`VERIFY_PROMPT`), the per-belief formatting (`format_belief_for_verify`), and the response parsing (`parse_verify_response`) are three independent pure functions. This makes each unit-testable in isolation (and indeed `tests/test_verify.py` exercises them separately).

- **`reasons/` vs `reasons_lib/` mirror.** The file tree shows an identical `reasons_lib/verify.py`. The `reasons` package is likely the in-repo implementation; `reasons_lib` is the published/installed variant. They should stay in lockstep—changes to one must be mirrored in the other.

## Dependencies

**Upstream (what this file imports):**

- `json`, `re`, `pathlib.Path` — stdlib only.
- `reasons.check_stale.resolve_source_path` — path-resolution logic shared with the stale-check pipeline. This couples verify to check_stale's path conventions.
- `reasons.llm.invoke_model` — the single LLM call boundary. All model interaction funnels through this one function.

**Downstream (what imports this):**

- `reasons/api.py` — the HTTP/RPC API layer that exposes a `/verify` endpoint.
- `reasons_lib/verify.py` — the library mirror (see above).
- `tests/test_verify.py` — the unit/integration tests.

Nothing else in the `reasons/` package imports `verify` directly. It's a leaf module with exactly one caller in production code.

## Flow

```
api.py  (or CLI)
  │
  ▼
verify_beliefs(beliefs_with_sources, model, timeout)
  │
  ├─ for each (belief, src) in beliefs_with_sources:
  │    format_belief_for_verify(belief, src)   ← pure string build
  │
  ├─ VERIFY_PROMPT.format(beliefs=beliefs_text) ← single big prompt
  │
  ├─ invoke_model(prompt, model, timeout)       ← one LLM round-trip
  │
  ├─ parse_verify_response(raw_response)        ← regex → JSON → validate
  │
  ▼
{ "belief-id-1": {"verdict":"CONFIRMED","reason":"...","quote":"..."},
  "belief-id-2": {"verdict":"STALE","reason":"...","quote":null},
  ... }
```

There is **no batching logic**—all beliefs are sent in a single LLM call. If you have 200 beliefs with 30 k-char sources each, that's a very large prompt. The truncation cap is the only safeguard.

`read_source` is *not* called inside `verify_beliefs`; the caller (e.g., `api.py`) is expected to have already resolved and read the sources and passed them in as the second element of each tuple. This separation lets the caller control caching, batching, and error-reporting of missing sources.

## Invariants

1. **Truncation is always applied.** Source content exceeding 30 000 chars is cut. The LLM never sees more than that per source.
2. **Verdicts are normalised to uppercase** in `parse_verify_response` (`.upper()`), regardless of what the LLM returns.
3. **Every valid entry in the response must have a string `"verdict"` key.** Entries missing that key, or where `verdict` is not a string, are silently dropped.
4. **`quote` is nullable.** A `null` quote is legal (e.g., for `INCONCLUSIVE`); a missing key defaults to `None` via `.get("quote")`.
5. **One LLM call per invocation of `verify_beliefs`.** There is no internal retry, no chunking, no fallback model.

## Error Handling

This module follows a **total-function / no-raise** discipline:

| Failure mode | What happens |
|---|---|
| Source file not found / unreadable | `read_source` returns `None`; the belief is formatted with `"(source not available)"`; the LLM should return `INCONCLUSIVE`. |
| LLM returns empty / `None` | `parse_verify_response` short-circuits to `{}`. |
| LLM returns non-JSON prose | Regex finds nothing → `{}`. |
| JSON parses but a value isn't a dict or lacks `"verdict"` | That entry is skipped; other valid entries survive. |
| `json.loads` throws | Caught; returns `{}`. |
| `read_text` throws `OSError` / `UnicodeDecodeError` | Caught; returns `None`. |

**What's notable:** there is no logging. If `invoke_model` raises (network timeout, auth error), that exception *will* propagate up to the caller—this module does not catch it. So the "no-raise" guarantee applies to the file-reading and response-parsing paths, but **not** to the LLM call itself. A caller in `api.py` must be prepared to handle that.

---

## Topics to Explore

- [file] `reasons/check_stale.py` — Contains `resolve_source_path`, the path-resolution logic that `read_source` depends on. Understanding its conventions (relative vs. absolute, `db_dir` semantics) is essential for debugging "source not found" issues.
- [file] `reasons/llm.py` — The single LLM invocation boundary (`invoke_model`). Knowing its retry, timeout, and error behaviour is critical since `verify_beliefs` does not wrap it in its own try/except.
- [file] `reasons/api.py` — The production caller. Shows how `verify_beliefs` is wired into the HTTP API, how sources are pre-read and cached, and how a `{}` return is surfaced to the end user.
- [file] `tests/test_verify.py` — The test suite. Reveals the expected edge cases (malformed JSON, missing source, truncation boundary) and the exact shape of the API contract.
- [general] `reasons-vs-reasons-lib-duplication` — The repo maintains parallel `reasons/` and `reasons_lib/` trees. Understanding the build/publish pipeline (check `pyproject.toml`, `.forge/code`) is necessary to avoid drift between the two copies of `verify.py`.

## Beliefs

- `verify-beliefs-sends-one-llm-call` — `verify_beliefs` makes exactly one `invoke_model` call regardless of how many beliefs are passed; there is no internal batching, chunking, or retry.
- `read-source-returns-none-never-raises` — `read_source` catches `OSError` and `UnicodeDecodeError` and returns `None` on failure; it never propagates an exception to the caller.
- `parse-verify-response-drops-malformed-entries` — Individual entries in the LLM JSON response that lack a string `"verdict"` key are silently discarded; only valid entries appear in the returned dict.
- `verify-beliefs-requires-preloaded-sources` — The caller must resolve and read source files before invoking `verify_beliefs`; the function itself never calls `read_source` or touches the filesystem.
- `reasons-verify-and-reasons-lib-verify-must-stay-in-sync` — Two copies of this module exist in the repo (`reasons/verify.py` and `reasons_lib/verify.py`); changes to one must be mirrored in the other to prevent divergence between the in-repo and library builds.
