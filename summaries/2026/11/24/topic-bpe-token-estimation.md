---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Explains why `chars / 4` is only a rough approximation and where it fails.

## Explanation

`reasons/compact.py:13-15` defines `estimate_tokens` as:

```python
def estimate_tokens(text: str) -> int:
    """Token estimate using chars/4 heuristic (standard BPE approximation)."""
    return max(1, len(text) // 4)
```

The same formula is duplicated in `reasons/pg.py:1929` and `reasons/pg.py:1940`, and `reasons/compact.py:73-74` uses `_char_count // 4` for `_current_tokens`. The token-budgeted summary docstring at `reasons/compact.py:32` explicitly calls this a `chars/4 BPE approximation`, while `tests/test_compact.py:109` notes that the budget is approximate. `tests/test_compact.py:12` even asserts the heuristic directly.

This is only rough because it assumes a fixed average of four characters per token. BPE tokenization is model- and vocabulary-specific: tokens are learned subword units, not characters. The same text can tokenize differently across tokenizers, and even within one tokenizer the ratio changes with language, punctuation, whitespace, numbers, code, emoji, and rare words. Character count cannot see token boundaries.

Where it fails in this code:

- Non-English / Unicode text: CJK, Cyrillic, Arabic, etc. often consume more tokens per character than English, so `len(text) // 4` can undercount badly.
- Code, markdown, IDs, URLs, and punctuation-heavy text: these often produce many short tokens, so `len(text) // 4` can underestimate the real token count.
- Short or empty text: `max(1, len(text) // 4)` returns `1` for empty and very short strings, so it is not a literal floor of `len(text) // 4`.
- Budget enforcement: `reasons/compact.py:64` reserves footer tokens via `estimate_tokens`, and `reasons/compact.py:73-74` derives current tokens from `_char_count`, so approximation errors can cause premature truncation or budget overrun.
- Duplicated logic: `reasons/pg.py:1929` and `reasons/pg.py:1940` repeat the same heuristic, so the approximation is not isolated to one module.

The observations do not show any actual tokenizer call or model vocabulary. That missing piece is what would be needed to quantify the error rate or replace the heuristic with exact token counts.

## Topics to Explore

- [file] `reasons/compact.py` — Inspect how `estimate_tokens` and `_current_tokens` drive budget enforcement and footer reservation.
- [function] `reasons/compact.py:estimate_tokens` — Compare the `chars/4` heuristic against a real tokenizer for the target models.
- [file] `reasons/pg.py` — Check the duplicated `max(1, len(text) // 4)` logic around lines 1929 and 1940.
- [file] `tests/test_compact.py` — See how the tests codify the heuristic and acknowledge approximation at line 109.
- [general] `tokenizer-choice` — Determine which tokenizer the budget is meant to approximate, since `chars/4` error depends heavily on the model vocabulary.

## Beliefs

- `estimate-tokens-uses-integer-division` — `estimate_tokens` returns `max(1, len(text) // 4)` in `reasons/compact.py:15`, so it is a character-count heuristic rather than a tokenizer call.
- `compact-budget-is-approximate` — The budget docstring at `reasons/compact.py:32` labels the token budget as a `chars/4` BPE approximation.
- `pg-duplicates-compact-heuristic` — `reasons/pg.py:1929` and `reasons/pg.py:1940` repeat the same `max(1, len(text) // 4)` estimation logic.
- `tests-codify-chars-div-4` — `tests/test_compact.py:12` asserts `estimate_tokens(text) == len(text) // 4`, locking in the heuristic behavior.
- `current-token-tracking-uses-char-count` — `reasons/compact.py:73-74` derives current tokens from `_char_count // 4`, enabling O(1) tracking instead of rejoining text.
