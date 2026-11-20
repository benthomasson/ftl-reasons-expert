---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Explore why the budget is approximate and how structural lines affect token estimates.

## Why the budget is approximate

The core budget logic lives in `reasons/compact.py`. The public `compact()` entry point accepts `budget` and documents it as a **“Maximum token budget (chars/4 BPE approximation)”** (`reasons/compact.py:32`). The approximation comes from `estimate_tokens()`:

```python
def estimate_tokens(text: str) -> int:
    """Token estimate using chars/4 heuristic (standard BPE approximation)."""
    return max(1, len(text) // 4)
```

See `reasons/compact.py:13-15`.

That means the code never runs the target model’s tokenizer. It assumes roughly one token per four characters, which is often close for English prose but can diverge for code, IDs, punctuation, markdown, whitespace, and model-specific BPE vocabulary. The budget is therefore a soft target, not an exact token limit. The test suite acknowledges this directly: `tests/test_compact.py:109` notes that the budget is approximate and mentions structural lines.

Inside `compact()`, accounting is done by tracking characters rather than tokens:

- `footer_tokens` is reserved up front at `reasons/compact.py:64`.
- `_char_count` starts with the initial title/header lines at `reasons/compact.py:66`.
- `_add_line()` appends text and increments `_char_count` by `1 + len(line)` to account for the newline separator (`reasons/compact.py:68-71`).
- `_current_tokens()` converts the accumulated character count with `max(1, _char_count // 4)` (`reasons/compact.py:73-74`).
- `_over_budget(line)` checks current estimated tokens plus the candidate line plus the reserved footer (`reasons/compact.py:76-77`).

This is fast and avoids repeatedly rejoining the output, but it inherits the chars/4 approximation and integer flooring. It also reserves the footer using a placeholder string:

```python
footer_tokens = estimate_tokens(f"Token count: ~{budget} / {budget} budget")
```

See `reasons/compact.py:64`. The actual footer later uses `_current_tokens()`, so the reserved footer size can differ slightly from the final footer if the current token count has a different number of digits or tokenizes differently.

The API and CLI both expose this same budget concept: `reasons/api.py:2869` defines `compact(budget: int = 500, ...)`, and the CLI parser exposes `--budget` with default `500` for the `compact` subcommand at `reasons/cli.py:3549`. The API wrapper also has a budget check at `reasons/api.py:2892`, but the core chars/4 estimator shown here is in `reasons/compact.py`.

## How structural lines affect token estimates

Structural lines are not belief content, but they still consume budget. Examples in `compact()` include:

- the title line,
- the stats header line,
- blank separators,
- section headers like `## Nogoods`, `## OUT (retracted)`, and `## IN (active)`,
- omission messages like `... (N more OUT nodes omitted)`,
- the hidden-by-summaries line,
- the final `Token count: ~... / ... budget` footer.

These lines are added through `_add_line()` and therefore count against `_char_count`. So section headers and blank separators reduce how much room remains for actual beliefs.

The bigger issue is that some structural lines are added **after** or **outside** the `_over_budget()` check:

- Section headers are checked before being added, e.g. the `if ... and not _over_budget("## Nogoods")` pattern around the nogoods block.
- But the blank line after each section is appended unconditionally when the section is present.
- When an item overflows, the code appends `... (N more ... omitted)` without checking whether that omission line itself fits.
- The hidden-by-summaries line is added after the IN loop without a budget check.
- The final footer is appended unconditionally, relying on the earlier `footer_tokens` reservation.

There is also a smaller accounting gap: `_over_budget(line)` estimates the candidate line with `estimate_tokens(line)`, but `_add_line()` later adds `1 + len(line)` to account for the newline. The candidate-line check does not include that extra newline character, so over many lines the accumulated newline characters can push the real estimate slightly past the check. Integer flooring in `_current_tokens()` can also hide or expose a fraction of a token.

So the budget is approximate for two compounding reasons:

1. **Token count is estimated as chars/4**, not measured with the actual tokenizer.
2. **Structural lines consume budget, but some are appended without a matching pre-check**, so the final output can exceed the nominal budget by a few tokens.

That is why the test at `tests/test_compact.py:109` explicitly treats the budget as approximate and calls out structural lines. The implementation is best understood as a priority-ordered, best-effort context summarizer: it tries to stay near the budget, not to enforce it exactly.

The observations do not include the full `tests/test_compact.py`, so the exact tolerance assertions are not visible here. The full `reasons/api.py` compact wrapper is also not shown in enough detail to describe its internal budget loop beyond the grep hits, but the core estimator and structural-line behavior are clear from `reasons/compact.py`.

## Topics to Explore

- [file] `reasons/compact.py` — Core budget accounting, section prioritization, and structural-line handling.
- [function] `reasons/compact.py:estimate_tokens` — The chars/4 heuristic and `max(1, ...)` behavior that make the budget approximate.
- [file] `tests/test_compact.py` — Exact test expectations around approximate budgets and structural lines, especially around line 109.
- [function] `reasons/api.py:compact` — The public API wrapper and how it threads `budget` into the network compact implementation.
- [general] `bpe-tokenization` — Why chars/4 diverges from real model tokenizers for code, IDs, and markdown.

## Beliefs

- `token-estimation-uses-chars-per-four` — `estimate_tokens` returns `max(1, len(text) // 4)` and is used to approximate token cost in `reasons/compact.py`.
- `compact-budget-includes-structural-lines` — Section headers, blank separators, omission messages, and the footer are added through `_add_line` and therefore consume budget.
- `over-budget-check-omits-newline-for-next-line` — `_over_budget` estimates a candidate line without the `+1` newline that `_add_line` later adds.
- `footer-reservation-uses-placeholder-budget` — `footer_tokens` is estimated from `"Token count: ~{budget} / {budget} budget"` rather than the final `_current_tokens()` value.
- `omission-lines-bypass-budget-check` — Overflow messages like `... (N more ... omitted)` are appended without an `_over_budget` check.
