---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Why chars/4 diverges from real model tokenizers for code, IDs, and markdown.

## Why chars/4 Diverges from Real Tokenizers for Code, IDs, and Markdown

### The assumption behind chars/4

Both estimators in this codebase are literal `max(1, len(text) // 4)`:

- `reasons/compact.py:13-15` — `estimate_tokens(text)` returns `max(1, len(text) // 4)`
- `reasons/pg.py:1928-1929` — duplicate `_estimate_tokens(text)` with the same formula

The docstring at `reasons/compact.py:14` calls this the *"standard BPE approximation."* It is only an approximation, and it is calibrated against **ordinary English prose**. Under byte-pair encoding (BPE, used by GPT/Claude/etc.), common English words and word-fragments are frequent merge candidates, so a well-trained vocabulary lands around **~4 characters per token** on prose. That average is the entire basis of the formula — and it breaks down the moment the input stops looking like English.

### Where it diverges

**Code.** Source code has a very different byte distribution than prose:
- Identifiers like `_estimate_tokens`, `_current_tokens`, `footer_tokens` (visible at `reasons/compact.py:64-77`) are long, snake_case or camelCase strings. BPE splits them at `_`, case boundaries, and rare subwords, so real tokenizers often emit **2–3 chars/token** for code — meaning the true token count is ~1.3–2× the chars/4 value.
- Punctuation, brackets, operators, and whitespace are hostile to the "long English word" prior. Indentation and repeated `_` / `: ` / `.` sequences usually tokenize as single characters or pairs, so dense code skews toward ~1 char/token.
- Comment/docstring English looks like prose and does hit ~4 chars/token — meaning a single file can have wildly different local ratios that chars/4 averages away.

**IDs, UUIDs, hashes, and URLs.** This is the worst case for chars/4, and it shows up directly in your tests:
- `tests/test_roundtrip.py:68,74,81,109` pass `accepted_pr="https://github.com/org/repo/pull/42"` around, and `tests/test_roundtrip.py:78,119` render it into markdown. That URL is ~38 characters but is typically **one token per character-ish chunk** — real tokenizers see maybe 25–35 tokens, while chars/4 predicts only 9–10. That's a **3–4× undercount for a single field**.
- The same applies to any UUID, git SHA, ULID, file path, or symbol hash. BPE vocabularies rarely contain these as merges because they're rare in training data, so they fall back to sub-word/character-level merges. Chars/4 was never designed to model this regime.

**Markdown.** Markdown adds structural overhead that is nearly invisible to chars/4:
- Fences, bullet markers (`- `), heading markers (`#`, `##`), backticks, and inline-code delimiters are typically emitted as **standalone or short tokens** because they are over-represented in training data as their own symbols.
- Inline code and fenced code blocks re-inject the "code" regime from above.
- A line like `### Heading` is 11 chars → chars/4 says 2 tokens; a real tokenizer often emits 4–5 (one per `#`/space sub-piece plus the merged word).
- Critically, your own `find_markdown_token_handling` search returned **zero matches** in the observations, so there is no whitespace/backtick/code-fence awareness anywhere in the estimator pipeline — it is text-in, `len//4`-out.

### Why this matters for this codebase, not just in theory

The estimator is used to enforce a compaction budget:

- `reasons/compact.py:74,77` — `_current_tokens()` + `estimate_tokens(line)` + `footer_tokens > budget` decides which lines get dropped.
- `reasons/pg.py:1940,1943` — same logic duplicated.
- Both emit a footer at `reasons/compact.py:164` and `reasons/pg.py:2029` showing `"Token count: ~{_current_tokens()} / {budget} budget"`.

If those lines include code snippets, PR URLs (as in `tests/test_roundtrip.py`), or markdown structure, the estimator undercounts and the compaction output exceeds the real context window even though it *claims* to fit the budget. The tests bake this in: `tests/test_compact.py:12` asserts `estimate_tokens(text) == len(text) // 4`, `tests/test_compact.py:17` asserts `estimate_tokens("hi") == 1`, `tests/test_compact.py:21` asserts an exact multiple. So the heuristic is enforced, not detected — any change toward a real tokenizer will break these tests.

### What the observations don't tell us

- The real `reasons/llm.py` is only shown via the shim `reasons_lib/llm.py` (3 lines: `from reasons.llm import *`), so we cannot confirm whether the LLM invocation path does any token counting or truncation before sending to the model. `find_token_estimator_symbols` only surfaced the compact/pg estimators, so it looks like no tokenizer is imported anywhere.
- `find_markdown_token_handling` returned `total_count: 0`, so there is no evidence of any markdown/ID-aware handling — divergence is purely structural, not a bug in a missing branch.
- We don't have the `compact.py` body between line 32 (budget docstring) and line 64 to see whether any per-line type (code vs prose) weighting exists. The visible code suggests not.

Net: chars/4 is a **prose-calibrated mean**, and code/IDs/markdown are systematically below that mean. The estimator isn't buggy per se — it's simply answering a different question than the real tokenizer.

## Topics to Explore

- [file] `reasons/compact.py` — Confirm the full line-selection loop and whether any content-type-aware weighting exists between lines 32 and 164
- [file] `reasons/pg.py` — Compare the duplicate estimator and budget check at lines 1928-1943 against `compact.py` for drift
- [general] `tokenizer-calibration` — Measure real chars/token ratios for this repo's code, test_roundtrip URLs, and emitted markdown to quantify the divergence empirically
- [file] `tests/test_compact.py` — These tests lock in the heuristic; any move to a real tokenizer requires updating lines 12, 17, and 21
- [general] `context-window-safety` — Decide whether budget overflow risk warrants a tokenizer-backed estimator in the compaction path

## Beliefs

- `estimate-tokens-is-chars-over-four` — `reasons/compact.py:15` and `reasons/pg.py:1929` both compute token counts as `max(1, len(text) // 4)`
- `estimate-tokens-has-no-zero` — The `max(1, ...)` wrapper means the estimator never returns 0, even for empty strings (`tests/test_compact.py:16`)
- `compact-and-pg-duplicate-estimator` — The chars/4 heuristic is implemented twice (`reasons/compact.py:13` and `reasons/pg.py:1928`) rather than shared
- `budget-check-requires-estimate-tokens` — Compaction cutoff at `reasons/compact.py:77` and `reasons/pg.py:1943` depends on the chars/4 estimate to decide which lines are dropped
- `estimate-tokens-is-locked-by-tests` — `tests/test_compact.py:12` asserts `estimate_tokens(text) == len(text) // 4`, so changing the formula is a test-breaking change
- `codebase-has-no-markdown-token-handling` — The `markdown|code block|identifier|UUID|hash|json` search returns zero matches, so no tokenizer-aware handling exists for code, IDs, or markdown
