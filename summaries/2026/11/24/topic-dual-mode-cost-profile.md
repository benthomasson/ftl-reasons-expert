---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: How `simple=True` vs `simple=False` changes 3 vs up to 5 LLM calls.

# How `simple=True` vs `simple=False` Changes 3 vs. up to 5 LLM Calls

## The contract, in one place

The authoritative description is the `ask()` docstring at **`reasons/ask.py:396-397`**:

> calls with simple=True (1 TMS + 1 FTS + 1 merge), or up to
> 5 with simple=False (up to 3 TMS tool-loop rounds + 1 FTS [ … ])

So the flag is a strategy switch between **fixed-pass synthesis** and an **agentic tool loop**, with a hard ceiling on LLM invocations in both cases.

## `simple=True` — exactly **3** LLM calls

- `simple=True` takes the `if simple:` branch at **`reasons/ask.py:433`**.
- It builds the prompt with `build_simple_prompt(question, beliefs_context, natural=natural)` at **`reasons/ask.py:450`**.
- `build_simple_prompt` is defined at **`reasons/ask.py:102`** and documented at **line 103** as:

  > "Build prompt for simple single-pass synthesis — **no tool definitions.**"

- The test `test_simple_prompt_has_no_tool_definition` at **`tests/test_ask.py:472-477`** confirms that in simple mode the prompt never includes tool definitions.

Because there are no tools and no loop, the budget is fixed:

| Call | Purpose |
|---|---|
| 1 | **TMS** — structured belief-network query |
| 2 | **FTS** — full-text search over belief texts |
| 3 | **merge** — single synthesis pass that turns both result sets into the answer |

Total = **3 LLM calls, deterministically**.

## `simple=False` — up to **5** LLM calls

- This is the default — the signature is `ask(..., simple=False, ...)` at **`reasons/ask.py:384`**.
- Instead of a single-pass prompt, the LLM is given tool definitions and runs an agentic loop. The docstring at **`reasons/ask.py:396-397`** describes the cap as **up to 3 TMS tool-loop rounds + 1 FTS [+ 1 merge]**.

| Call(s) | Purpose |
|---|---|
| up to 3 | **TMS tool-loop rounds** — each round is an LLM turn because the model decides what (if anything) to query next |
| 1 | **FTS** |
| 1 | **final synthesis / merge** |

Total = **up to 5 LLM calls**, with the actual count in the 3–5 range depending on how many TMS rounds the model takes.

## Why it matters

1. **Cost & latency ceiling** — `simple=True` is a hard 3-call budget; `simple=False` can hit 5.
2. **Determinism** — simple mode always takes the same retrieval-then-merge path; non-simple mode's call count is chosen by the model inside the tool loop.
3. **No tool-use in simple** — line 103 spells it out ("no tool definitions"), so the LLM cannot self-direct retrieval; TMS and FTS are prefetched before the single merge call.
4. **CLI wiring** — `--simple` is a `store_true` flag at **`reasons/cli.py:3596`** (opt-in) and is forwarded at **`reasons/cli.py:1374`** into `ask(simple=args.simple)`.

## What the observations don't show (gaps)

The following are **not** in the gathered snippets, so the exact mechanics of the 5-call path can't be fully verified here:

- The **`else:` branch of `if simple:`** (line 433) — the code that actually implements the TMS tool loop.
- How the **3-round TMS cap** is enforced (round counter, stop condition, etc.).
- Whether **TMS and FTS in `simple=True` are separate LLM calls or programmatic DB queries** — the docstring counts them as calls, implying LLM-formulated queries, but the code isn't shown.
- The **`build_final_prompt`** / `build_ask_prompt` functions referenced in the imports at `tests/test_ask.py:12`, which likely belong to the non-simple branch.

## Topics to Explore

- [function] `reasons/ask.py:ask` — Read the full `else:` branch after `if simple:` at line 433 to see the TMS tool-loop implementation and its 3-round cap.
- [function] `reasons/ask.py:build_simple_prompt` — Confirm whether TMS/FTS are prefetched programmatically or represent separate LLM calls in the simple path.
- [file] `reasons/ask.py` — Walk lines 384–500 for the complete control flow of both branches end-to-end.
- [file] `tests/test_ask.py` — Tests at lines 437–477 exercise simple-mode timeouts, errors, no-results, and confirm the no-tool-definition contract.
- [file] `reasons/cli.py` — Lines 1374 and 3596 show how `--simple` surfaces as a CLI flag and is plumbed into `ask()`.

## Beliefs

- `simple-flag-enables-single-pass-synthesis` — `simple=True` routes `ask()` through `build_simple_prompt`, which has no tool definitions, yielding exactly 3 LLM calls (TMS + FTS + merge).
- `simple-false-uses-agentic-tool-loop` — `simple=False` (the default) drives an agentic tool loop capped at 5 LLM calls (up to 3 TMS rounds + 1 FTS + 1 merge).
- `build-simple-prompt-has-no-tool-definitions` — Documented at `reasons/ask.py:102-103` and asserted by `test_simple_prompt_has_no_tool_definition`.
- `simple-flag-defaults-to-false` — `ask()`'s signature at `reasons/ask.py:384` and the `store_true` CLI flag at `reasons/cli.py:3596` both default `simple` to `False`.
- `simple-flag-is-forwarded-from-cli` — `reasons/cli.py:1374` forwards `args.simple` into `ask(simple=...)`.
