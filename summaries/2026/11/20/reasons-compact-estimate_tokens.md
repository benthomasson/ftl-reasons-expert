---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: estimate_tokens in reasons/compact.py

## Purpose

`estimate_tokens` provides a cheap, deterministic approximation of how many BPE tokens a string would consume. It exists so `compact` can make budget decisions without invoking a real tokenizer, which would be slower and model-dependent.

It uses the common `chars / 4` heuristic for English-like text and guarantees at least one estimated token for any string.

## Contract

- **Preconditions**
  - `text` is expected to be a `str`.
  - The type hint is not runtime-enforced; any object with a sensible `len()` may work.
  - Callers should treat the result as an approximation, not an exact model token count.

- **Postconditions**
  - Returns an `int`.
  - The return value is always `>= 1`.
  - The result depends only on `len(text)`, not on the text contents.
  - The function is deterministic and pure.

- **Invariants**
  - For any `str`, `result == max(1, len(text) // 4)`.
  - Lengths `0` through `7` all return `1`.
  - For longer strings, the result increases in steps of one every four characters, e.g. length `8 -> 2`, `12 -> 3`, etc.

- **Important approximation caveat**
  - This is not a real BPE tokenizer. It assumes roughly four characters per token.
  - The heuristic is language- and tokenizer-dependent.
  - Python `len(str)` counts Unicode code points, not bytes, grapheme clusters, or model tokens.

## Parameters

- `text: str`
  - The string whose token cost should be estimated.
  - Edge cases:
    - `""` returns `1`, not `0`.
    - Short strings of length `1` through `7` return `1`.
    - Non-ASCII text may be poorly estimated because code-point count does not correspond directly to model tokens.
    - Passing `None` triggers `TypeError` from `len(None)`.
    - Passing a list or tuple happens to work because it has `len()`, even though the annotation says `str`; the function does not validate the type.

## Return Value

Returns an integer estimate:

```python
max(1, len(text) // 4)
```

The caller must not interpret this as an exact token count. It is suitable for budgeting and rough sizing. It intentionally never returns zero, so every string has a minimum estimated cost of one token.

## Algorithm

1. Compute `len(text)`.
2. Perform integer floor division by `4`.
3. Clamp the result to a minimum of `1` using `max(1, ...)`.
4. Return the resulting integer.

That’s the entire implementation. There is no tokenization, no caching, no inspection of content, and no dependence on whitespace or punctuation.

## Side Effects

None. The function is pure: it does not mutate arguments, perform I/O, access global state, or call external services.

## Error Handling

There is no explicit validation or exception handling.

- For a normal `str`, it should not raise.
- For `None` or an object without `__len__`, `len(text)` raises `TypeError`.
- For a sequence such as a list, it may return a value based on element count even though the type hint says `str`.
- Any exception raised by a custom object’s `__len__` would propagate.
- The function never returns an error sentinel and never swallows exceptions.

## Usage Patterns

`estimate_tokens` is used inside `reasons/compact.py:compact` for token-budget accounting.

Key call sites:

- `footer_tokens = estimate_tokens(...)` reserves space for the final footer line.
- `_over_budget(line)` checks:

  ```python
  _current_tokens() + estimate_tokens(line) + footer_tokens > budget
  ```

- `_current_tokens()` duplicates the same heuristic over an accumulated character count:

  ```python
  max(1, _char_count // 4)
  ```

Important caller obligations:

- The `budget` passed to `compact` must be understood in the same heuristic token units.
- Callers should leave margin because the estimate can be inaccurate for non-English text, code, emoji, etc.
- Because `estimate_tokens("")` returns `1`, callers should not assume empty strings are free.
- In `compact`, candidate line estimates do not include the newline character directly, while `_add_line` tracks `1 + len(line)`. The budget check is therefore approximate and may be slightly optimistic around newline costs.

## Dependencies

The function itself depends only on Python built-ins:

- `len`
- `max`
- integer floor division `//`

It does not rely on any external tokenizer or module-level imports from `reasons/compact.py`.

## Topics to Explore

- [file] `reasons/compact.py` — Shows how `estimate_tokens` participates in the broader token-budgeted belief-state summary.
- [function] `reasons/compact.py:compact` — Demonstrates the actual budget checks, footer reservation, and section truncation logic.
- [file] `tests/test_compact.py` — Reveals the expected behavior around small budgets, truncation, and prioritization.
- [general] `bpe-token-estimation` — Explains why `chars / 4` is only a rough approximation and where it fails.
- [general] `token-budgeted-context-injection` — Covers the design pattern of fitting summaries into LLM context windows.

## Beliefs

- `estimate-tokens-uses-floor-division` — The function returns `max(1, len(text) // 4)`, so lengths `0` through `7` all estimate to `1`.
- `estimate-tokens-is-pure` — It has no side effects and depends only on `len(text)`.
- `estimate-tokens-approximates-bpe-without-tokenizer` — It uses a character-count heuristic rather than an actual BPE tokenizer or model vocabulary.
- `compact-uses-estimate-tokens-for-budget-checks` — `reasons/compact.py:compact` uses it to reserve footer space and test candidate lines against the budget.
- `estimate-tokens-requires-sized-object` — Passing `None` or an object without `__len__` raises `TypeError` from `len`.
