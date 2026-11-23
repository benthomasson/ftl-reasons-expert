---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: String-matching the model’s refusal phrase is fragile; explore prompt/parse coupling invariants

# String-Matching the Model's Refusal Phrase Is Fragile

## What the observations actually show

The search for refusal-related literals in production code turned up **nothing meaningful**. The `find_refusal_related_tests` sweep (`refus|sorry|cannot|unable`) returned `total_count: 0`, and the only real refusal-phrase hits are inside test fixtures:

- `tests/test_ask.py:182` and `:190` — both define `refusal = "I don't have enough beliefs in the network to answer this question."` and patch `reasons.ask.invoke_model` to return it.
- `tests/test_api.py:511` — patches `reasons.llm.invoke_model` with `"Sorry, I cannot do that."`

Meanwhile, the prompt/parse scan shows the LLM layer is entirely text-passthrough:

- `reasons/llm.py:123-137` — `_parse_cli_json` only pulls `data["result"]` (claude) or `data.get("response", output)` (gemini). No refusal logic.
- `reasons/llm.py:202-246` — `_invoke_api` returns `response.content` verbatim.
- `reasons/llm.py:275-320` — `invoke_model` just dispatches to CLI/API paths.

So the application code never inspects output for a refusal phrase. If a refusal contract exists at all, it lives in `reasons/ask.py` downstream of `invoke_model`, and we cannot see it in the observations.

## Why this coupling is fragile

The implied invariant is something like:

1. `reasons/ask.py` builds a prompt telling the model to refuse with a specific sentence when it lacks evidence.
2. The model emits that sentence (or something close to it).
3. Some layer matches the sentence to change behavior (e.g., return an empty answer, not a fabricated one).

Every leg of that invariant is weak:

- **No shared constant.** The phrase `"I don't have enough beliefs in the network to answer this question."` exists only as a Python string literal in `tests/test_ask.py:182/190`. Nothing in production guarantees prompt text == test fixture == parser check.
- **Tests bypass the crucial seam.** By patching `reasons.ask.invoke_model`, the tests never exercise the prompt or `_parse_cli_json`. So they'd stay green even if the prompt stopped telling the model to refuse, or if the parser started stripping whitespace/punctuation that changes the comparison.
- **Two different refusal styles in tests.** `test_ask.py` uses the domain-specific phrase, while `test_api.py:511` uses `"Sorry, I cannot do that."` If production code really were string-matching, it would have to recognize both — a strong hint that no robust matching exists.
- **Natural-language output drifts.** Models rephrase ("I don't have sufficient beliefs…", "I'm not able to answer…", trailing periods dropped, etc.). Any `response == PHRASE` or `PHRASE in response` check is bound to the exact tokenization the test mocks, not the model's actual distribution.

## Better invariants to aim for

Instead of matching prose, make the contract **structural**, since `reasons/llm.py:_parse_cli_json` already parses JSON:

- Have the prompt require JSON output, e.g. `{"refused": true/false, "answer": "..."}`, and parse that flag in `_parse_cli_json` or in `reasons/ask.py`.
- Or embed a sentinel marker the parser can recognize deterministically (e.g., a leading `REFUSE:` prefix) and reject early.
- Or use a stop-sequence / tool-call channel so "refused" is a control-flow signal, not a string.

What's missing from the observations to fully judge this:

- The body of `reasons/ask.py` around where it calls `invoke_model` and what it does with the result.
- The prompt template (does it actually instruct the exact phrase?).
- The assertions in `tests/test_ask.py` after line 191 — do they check for empty output, an exception, or a specific return?

Without those, we can only say the current design depends on a natural-language string that nothing in the codebase enforces.

## Topics to Explore

- [function] `reasons/ask.py:invoke_model_callers` — Find the actual prompt template and see whether the refusal phrase is named in the prompt; that's the other half of the coupling invariant.
- [file] `reasons/llm.py` — Review `_parse_cli_json` (lines 123-137) and `_invoke_api` (lines 202-246) to confirm there is no refusal detection, then design a structured-output contract.
- [file] `tests/test_ask.py` — Read the full body of tests around lines 182-200 to see what behavior is being asserted versus what is only being mocked.
- [general] `structured-refusal-signal` — Compare JSON-field, sentinel-prefix, and stop-sequence approaches for making refusals machine-detectable instead of string-matched.
- [general] `prompt-parser-contract-tests` — Design cross-layer tests that exercise the prompt → parser → caller path end-to-end so phrase drift is caught in CI.

## Beliefs

- `refusal-phrase-is-test-only-fixture` — The refusal string `"I don't have enough beliefs in the network to answer this question."` appears only in `tests/test_ask.py:182/190`, not as a shared production constant.
- `refusal-detection-absent-in-llm-layer` — `reasons/llm.py` (including `_parse_cli_json`, `_invoke_api`, `invoke_model`) contains no refusal-phrase matching; the parser returns raw model text.
- `llm-layer-returns-raw-text` — `_parse_cli_json` (line 137) falls back to `data.get("response", output)` and `_invoke_api` (line 246) returns `response.content`, so refusals reach callers as plain strings.
- `tests-bypass-prompt-parser-seam` — Tests patch `reasons.ask.invoke_model` and `reasons.llm.invoke_model` directly, so neither the prompt nor `_parse_cli_json` is exercised.
- `refusal-contract-requires-ask-layer` — Because `reasons/llm.py` has no refusal logic, any refusal handling must live in `reasons/ask.py` (or above), coupled to a natural-language phrase.
