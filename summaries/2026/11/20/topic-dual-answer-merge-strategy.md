---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: How TMS and FTS answers are combined and why a third LLM call is used.

## How TMS and FTS answers are combined

Dual mode is answer-level fusion, not retrieval-level fusion. In `reasons/ask.py:383`, `ask(..., dual=True, sources_db=...)` runs two independent answer paths:

1. **TMS / belief-network path**
   - The top-level `ask()` calls itself for the TMS answer at `reasons/ask.py:408`.
   - This path uses the belief network prompt/tool-loop machinery, and with `simple=False` can use up to `MAX_ITERATIONS = 3` TMS tool-loop rounds.
   - The `ask()` docstring documents this at `reasons/ask.py:394-398`.

2. **FTS / source-document RAG path**
   - The top-level `ask()` calls `_fts_rag_answer(...)` at `reasons/ask.py:412`.
   - `_fts_rag_answer` is defined at `reasons/ask.py:322`, retrieves source chunks via `_search_source_chunks` / `search_source_chunks`, then synthesizes with `FTS_RAG_PROMPT` (`reasons/ask.py:268`).
   - `search_source_chunks` queries the `chunks_fts` FTS5 table and orders by rank (`reasons/ask.py:220-246`).

After both paths return, `ask()` checks whether either answer is effectively empty:

- TMS is treated as empty if it equals `NO_BELIEFS_MSG` or contains “don’t have enough beliefs” (`reasons/ask.py:414-415`).
- FTS is treated as empty if it starts with `"No relevant documents"` (`reasons/ask.py:416`).

If exactly one side is empty, the other answer is returned directly:

- TMS empty, FTS non-empty → return FTS answer (`reasons/ask.py:417-419`).
- FTS empty, TMS non-empty → return TMS answer (`reasons/ask.py:420-422`).

If both answers are substantive, `ask()` invokes `_merge_answers(...)` at `reasons/ask.py:426`.

## Why the third LLM call exists

The third call is a **semantic merge/reconciliation step**, implemented by `_merge_answers` at `reasons/ask.py:331` using `MERGE_PROMPT` (`reasons/ask.py:301-320`).

It is needed because the two answers are produced independently from different sources and prompts:

- Answer A comes from a structured belief network with dependency chains.
- Answer B comes from full-text search over source documents.
- They may overlap, duplicate points, use different levels of specificity, or contradict each other.
- They also preserve different citation styles: belief IDs in `[brackets]` for TMS, and `[filename]` references for FTS.

`MERGE_PROMPT` explicitly asks the model to:

- combine information from both answers,
- prefer the more specific/detailed version when both cover the same point,
- preserve both citation styles,
- avoid adding information not present in either answer,
- note disagreements when the answers contradict.

So the third call is not a third retrieval step. It is an LLM-based final synthesis step that turns two independently generated answers into one coherent answer. It is conditional: when one side has no evidence, the code skips the merge call and returns the non-empty answer directly. With `simple=True`, dual mode makes exactly three LLM calls: TMS, FTS, merge. With `simple=False`, it can make up to five: up to three TMS tool-loop rounds, one FTS call, and one merge call (`reasons/ask.py:394-398`).

The provided excerpt does not show the both-empty branch; it shows the one-side-empty shortcuts and the both-non-empty merge path.

## Topics to Explore

- [file] `reasons/ask.py` — Full dual-mode orchestration, prompt templates, and empty-answer handling.
- [function] `reasons/ask.py:_merge_answers` — Builds `MERGE_PROMPT` and invokes the third LLM call.
- [function] `reasons/ask.py:_fts_rag_answer` — FTS retrieval plus synthesis path that produces Answer B.
- [function] `reasons/ask.py:ask` — Top-level dual-mode branching and recursive TMS call.
- [general] `dual-mode-cost-profile` — How `simple=True` vs `simple=False` changes 3 vs up to 5 LLM calls.

## Beliefs

- `dual-mode-uses-three-llm-calls-with-simple-true` — In dual mode with `simple=True`, the code makes one TMS call, one FTS RAG call, and one merge call.
- `merge-call-is-skipped-when-one-answer-is-empty` — If either the TMS or FTS answer is empty, `ask()` returns the non-empty answer directly instead of calling `_merge_answers`.
- `merge-prompt-preserves-both-citation-styles` — `MERGE_PROMPT` instructs the model to preserve TMS belief-ID citations and FTS filename citations.
- `fts-rag-returns-no-relevant-documents-sentinel` — `_fts_rag_answer` returns `"No relevant documents found for this question."` when source-chunk retrieval returns no context.
- `tms-tool-loop-can-increase-dual-calls-to-five` — With `simple=False`, dual mode can use up to three TMS tool-loop rounds plus one FTS call plus one merge call.
