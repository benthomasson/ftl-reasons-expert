---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: How the `dual=True` path combines TMS and FTS answers, of which this chunk context is one input

## How the `dual=True` path combines TMS and FTS answers

### Precondition and trigger

`ask()` is defined at `reasons/ask.py:383`. The `dual` parameter is documented at `reasons/ask.py:394`:

> "dual: run TMS and FTS RAG in separate calls, then merge the two in a third call. Requires sources_db. Makes 3 LLM calls with simple=True (1 TMS + 1 FTS + 1 merge), or up to 5 with simple=False (up to 3 TMS tool-loop rounds + 1 FTS + 1 merge). Timeout applies per call."

The dual path is guarded by two lines (`reasons/ask.py:403–406`):

- `403: if dual and not sources_db:` → `404: raise ValueError("--dual requires --full-sources")`
- `406: if dual and sources_db:` → the dual branch itself begins here

The body of that branch (from `ask.py:407` onward) is **not shown** in the observations — the content inspection stops inside `ask()`'s docstring. What the branch does is therefore inferred from three reliable sources: the docstring above, the three helper functions it must use, and the tests that assert its call count and preconditions.

### The three-phase data flow

The dual path stitches together two independent retrievers and a final LLM merger:

```
question ──┬─▶ TMS pipeline (belief FTS + synthesis) ─▶ answer_tms ─┐
           │                                                         ├─▶ _merge_answers ─▶ merged answer
           └─▶ _fts_rag_answer ─▶ _search_source_chunks ─▶ chunks_fts │
                             ─▶ FTS_RAG_PROMPT ─▶ answer_fts ─────────┘
```

#### Phase A — Answer A: the belief-network (TMS) answer

This is the normal, non-dual `ask()` pipeline. It runs FTS5 search over the belief database (see `api.py:3169–3173` for the `nodes_fts MATCH` query and `_fts_query`), builds a prompt with `build_ask_prompt` / `build_simple_prompt` / `build_final_prompt` (`ask.py:130–219`), and — when `simple=False` — enters the tool loop bounded by `MAX_ITERATIONS = 3` (defined near `ask.py:347`). The result is `answer_tms`.

#### Phase B — Answer B: the source-document (FTS) answer

This is where **the chunk context** enters. `_fts_rag_answer(question, sources_db, model, timeout)` at `reasons/ask.py:322–329`:

1. Calls `_search_source_chunks(question, sources_db)` (`ask.py:252–266`), which wraps `search_source_chunks()` (`ask.py:220–249`). That function:
   - Tokenizes the question with `re.findall(r'\w+', query)`, drops stop words and 1-letter tokens, falls back to length > 1 tokens if the filter empties out (`ask.py:226–229`).
   - Builds `fts_query = " OR ".join(f'"{w}"' for w in words)` (`ask.py:232`).
   - Runs `FROM chunks_fts JOIN chunks c ON c.id = chunks_fts.rowid WHERE chunks_fts MATCH ? ORDER BY chunks_fts.rank LIMIT ?` (`ask.py:236–245`).
   - Returns rows plus filename/section metadata.
2. Scopes the result via `_search_source_chunks`, which formats each chunk as `### [i] filename > section\n\n<text>` joined by `\n\n---\n\n` (`ask.py:256–265`).
3. If there is no context, returns the literal sentinel `"No relevant documents found for this question."`.
4. Otherwise formats `FTS_RAG_PROMPT` (`ask.py:268–285`) with `{context}` = the formatted chunks and `{question}`.
5. Calls `invoke_model(prompt, model=model, timeout=timeout).strip()`.

So the "chunk context" is the **`{context}` slot inside `FTS_RAG_PROMPT`** — the raw source excerpts are *not* forwarded to the merge step; only the synthesized `answer_fts` (Answer B) is.

#### Phase C — Merge: `_merge_answers`

`_merge_answers(question, answer_tms, answer_fts, model, timeout)` (~`ask.py:331–338`) formats `MERGE_PROMPT` and calls `invoke_model` directly:

- `{answer_tms}` at `ask.py:314` = Answer A
- `{answer_fts}` at `ask.py:318` = Answer B
- The prompt instructs the merger to:
  - Combine information from both answers.
  - Prefer the more specific/detailed version when both cover the same point.
  - Preserve **both citation styles**: belief IDs in `[brackets]` from Answer A and `[filename]` from Answer B.
  - Not add information neither answer contains.
  - Explicitly note contradictions.

The merge result is returned from `ask()` (with `.strip()`).

### Contract asserted by tests

The observations confirm three invariants that the dual branch must honor:

| Test | File:line | What it pins down |
|---|---|---|
| `test_dual_calls_three_times` | `tests/test_ask.py:792, 808` | With `simple=True, dual=True`, exactly 3 `invoke_model` calls (1 TMS + 1 FTS + 1 merge) |
| `test_dual_without_sources_db_raises` | `tests/test_ask.py:813–818` | `dual=True` without `sources_db` raises `ValueError("--dual requires --full-sources")` |
| `test_dual_propagates_natural` | `tests/test_ask.py:820, 837` | The `natural=True` flag reaches the sub-calls (stripping IDs/status from belief context via `_strip_belief_metadata` at `ask.py:166–208`) |

### What's missing / inferred

The literal body of `if dual and sources_db:` (`ask.py:406+`) was not included in the observations, so the following are inferred, not observed directly:

- Whether the TMS-side call reuses the identical `_beliefs_or_no_match` / `NO_BELIEFS_MSG` short-circuit logic (`ask.py:348–355`) before invoking `_merge_answers`.
- The exact propagation of `model`, `timeout`, and `natural` into `_fts_rag_answer` and `_merge_answers` (the test only tells us `natural` is propagated somewhere).
- Whether `_strip_belief_metadata` is applied to `answer_tms` before it becomes `{answer_tms}` in the merge when `natural=True`.
- Whether the merger is skipped when either side returns a "no results" sentinel.

To fully close the loop, read `reasons/ask.py:406–566` and `tests/test_ask.py:780–840`.

## Topics to Explore

- [file] `reasons/ask.py` — Read lines ~406–566 for the actual dual branch body (currently only the precondition at line 406 is visible)
- [function] `reasons/ask.py:_merge_answers` — See the exact `MERGE_PROMPT.format(...)` call and `.strip()` semantics
- [function] `reasons/ask.py:_fts_rag_answer` — Traces the chunk-context flow that produces Answer B
- [function] `tests/test_ask.py:test_dual_calls_three_times` — Pins the 3-call (`simple=True`) contract; also shows what `dual` short-circuits or preserves
- [general] `natural-flag-propagation` — How `natural=True` is threaded through the TMS sub-call, FTS sub-call, and merge in the dual path

## Beliefs

- `dual-requires-sources-db` — `ask(dual=True)` raises `ValueError("--dual requires --full-sources")` when `sources_db` is `None`, enforced at `reasons/ask.py:403–404` and asserted by `tests/test_ask.py:813–818`.
- `dual-simple-makes-three-llm-calls` — With `simple=True`, `dual=True` performs exactly 3 `invoke_model` calls (TMS synthesis, FTS RAG synthesis, merge), per the docstring at `reasons/ask.py:394–398` and `tests/test_ask.py:792`.
- `chunk-context-feeds-fts-rag-prompt` — Source-document chunks returned by `_search_source_chunks` are inserted as `{context}` inside `FTS_RAG_PROMPT` (`reasons/ask.py:268–285`); they are not passed directly to `MERGE_PROMPT`.
- `merge-prompt-preserves-both-citations` — `MERGE_PROMPT` (with `{answer_tms}` at line 314 and `{answer_fts}` at line 318) explicitly instructs the model to preserve `[belief-id]` citations from Answer A and `[filename]` citations from Answer B.
- `merge-and-fts-steps-call-invoke-model` — Both `_fts_rag_answer` (`reasons/ask.py:322–329`) and `_merge_answers` call `invoke_model(...).strip()` directly, so `timeout` in the dual path is enforced per-call rather than as a global budget.
