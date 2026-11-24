---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: _merge_answers in reasons/ask.py

# `_merge_answers` — `reasons/ask.py`

## 1. Purpose

`_merge_answers` is the third and final step of the **dual-retrieval path** in `ask()`. When the caller requests `dual=True` with a `sources_db`, `ask()` runs two independent answer pipelines:

- **TMS path** (`ask()` recursively, hitting the belief network via `api.search` + LLM synthesis)
- **FTS RAG path** (`_fts_rag_answer`, hitting the source-document FTS index)

Once both answers exist and neither is a "no results" sentinel, `_merge_answers` sends them together to the LLM to produce a single coherent answer. It exists to prevent the caller from having to choose between the two answers or concatenate them naively — the LLM is used to deduplicate overlapping points and surface contradictions while preserving citations from both sides.

## 2. Contract

**Preconditions** (enforced by the caller, not by the function):
- `answer_tms` and `answer_fts` are both non-empty strings representing *real* answers — the caller has already checked `tms_empty` and `fts_empty` and returned early otherwise.
- `MERGE_PROMPT` exists in module scope and contains `{question}`, `{answer_tms}`, `{answer_fts}` placeholders (it does).
- `invoke_model` is imported and returns a string.

**Postconditions:**
- Returns a whitespace-stripped string from the LLM. No structural validation of the returned content.
- Does not mutate any argument.

**Invariants:**
- One LLM call per invocation.
- No fallback: if `invoke_model` fails, the error propagates (see Error Handling).

## 3. Parameters

| Parameter | Type | Meaning | Edge cases |
|---|---|---|---|
| `question` | `str` | The original user question, interpolated into the prompt. | Empty string would still work but produces a nonsensical prompt; not validated. |
| `answer_tms` | `str` | Answer from the belief-network pipeline ("Answer A"). | Caller guarantees non-empty. May contain `[belief_id]` citations and status/justification metadata if `natural=False`. |
| `answer_fts` | `str` | Answer from the FTS RAG pipeline ("Answer B"). | Caller guarantees non-empty. May contain `[filename]` citations per `FTS_RAG_PROMPT`. |
| `model` | `str` | Model identifier forwarded to `invoke_model`. | Default `"claude"`. Not validated or enumerated here. |
| `timeout` | `int`/`float` | Per-call timeout in seconds forwarded to `invoke_model`. | Default `300`. Note the docstring for `ask()` says timeout applies *per call*, so the merge adds another full `timeout` window. |

## 4. Return Value

- On success: `str` — the model's merge response, stripped of leading/trailing whitespace.
- On failure: **does not return** — exceptions propagate (see §7).
- The caller (`ask`) returns this value directly to its own caller.

There is no sentinel return value, no "insufficient information" fallback, and no error object. Callers must assume any string return is a well-formed answer text.

## 5. Algorithm

1. Format `MERGE_PROMPT` with `question`, `answer_tms`, `answer_fts`. The prompt frames the two answers as "Answer A (Belief Network)" and "Answer B (Source Documents)" and instructs the model to:
   - Combine both,
   - Prefer the more specific version on overlap,
   - Preserve `[belief IDs]` and `[filenames]`,
   - Not add information from outside the two answers,
   - Note contradictions.
2. Call `invoke_model(prompt, model=model, timeout=timeout)`.
3. `.strip()` the result and return it.

That's the entire logic — one prompt-format, one LLM call, one string trim.

## 6. Side Effects

- **One external LLM invocation** via `invoke_model` (subprocess/network, depending on `.llm`).
- No file, DB, or global state mutation.
- No logging inside `_merge_answers`; the caller prints `"Dual path: merging answers..."` to stderr *before* invoking this function.

## 7. Error Handling

**None.** Unlike the simple and tool-loop paths in `ask()` (which wrap `invoke_model` in `try/except` for `subprocess.TimeoutExpired` and `Exception` and fall back to showing raw context), `_merge_answers` lets everything propagate. Any of the following escapes to the caller:

- `subprocess.TimeoutExpired` if the merge call exceeds `timeout`
- `KeyError` / `IndexError` if `MERGE_PROMPT` is malformed (won't happen with current constant)
- `AttributeError` if `invoke_model` returned something without `.strip()` (e.g. `None`)
- Any exception raised inside `invoke_model` (network, subprocess, JSON parsing, etc.)

This is a behavioral asymmetry worth knowing: in the `dual` path, `_merge_answers` and `_fts_rag_answer` are the two functions that can raise out of `ask()`, whereas the TMS path (which recurses into `ask()`) catches its own errors.

## 8. Usage Patterns

Single call site in `ask()`:

```python
if tms_empty and not fts_empty:      return answer_fts
if fts_empty and not tms_empty:      return answer_tms
if tms_empty and fts_empty:          return NO_BELIEFS_MSG
print("Dual path: merging answers...", file=sys.stderr)
return _merge_answers(question, answer_tms, answer_fts, model=model, timeout=timeout)
```

**Caller obligations:**
- Only pass non-empty, real answers (the empty-case short-circuits above must run first).
- Be prepared for an exception to escape — callers of `ask(dual=True, ...)` may want their own `try/except` because this branch is not protected by `ask()`'s internal handlers.
- Passing `natural=True` upstream affects the *contents* of `answer_tms` (metadata is stripped in the TMS subcall), but `_merge_answers` itself does no natural-vs-structured handling — it just passes the strings through.

## 9. Dependencies

- `MERGE_PROMPT` — module-level constant in `reasons/ask.py`; defines the merge instructions and citation-preservation rules.
- `invoke_model` — imported from `.llm` (`from .llm import invoke_model`); performs the actual LLM request and is the sole source of possible exceptions and the returned text.
- Built-in `str.format` and `str.strip`.

No dependency on `api`, `sqlite3`, `json`, or MCP bridges — those are used elsewhere in the module.

## Assumptions not enforced by types

- Both answer strings are *substantive* answers (not the `NO_BELIEFS_MSG` sentinel or the `"No relevant documents found..."` string). Only convention (the caller's early-return checks) guarantees this.
- `model` is a model name that `invoke_model` understands — no enumeration or validation here.
- `timeout` is in seconds and is a separate budget per LLM call, not cumulative across the three dual-path calls.
- The LLM will honor the "do not add information" / "preserve citations" rules — nothing in this function verifies the merged answer afterwards.

---

## Topics to Explore

- [file] `reasons/ask.py` — The surrounding `ask()` pipeline, especially the `dual` branch and its empty-answer short circuits.
- [function] `reasons/ask.py:ask` — Orchestrates the TMS + FTS + merge flow and is the sole caller of `_merge_answers`.
- [function] `reasons/llm.py:invoke_model` — Defines what exceptions and return types `_merge_answers` must tolerate (or not).
- [file] `tests/test_ask.py` — Expected behavior of the dual path and merge prompt.
- [general] `dual-retrieval-strategy` — Trade-offs between merging two retrievers via LLM vs. concatenating or picking one.

## Beliefs

- `_merge-answers-requires-non-empty-inputs` — `_merge_answers` is only invoked after the caller has confirmed both TMS and FTS answers are non-empty (via `tms_empty`/`fts_empty` checks).
- `_merge-answers-propagates-exceptions` — Exceptions from `invoke_model` inside `_merge_answers` are not caught and escape `ask()` in the dual path.
- `_merge-answers-produces-single-answer` — Returns a single stripped string produced by one LLM call using `MERGE_PROMPT`.
- `_merge-answers-preserves-citations` — The merge prompt instructs the LLM to preserve `[belief IDs]` from Answer A and `[filenames]` from Answer B.
- `_merge-answers-depends-on-invoke-model` — The only external call is `invoke_model(prompt, model, timeout)`; no DB, filesystem, or MCP access occurs.
