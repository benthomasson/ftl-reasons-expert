---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Map which functions contain the `with _cost_lock` sites and whether any locked path calls another locked path.

All observed `with _cost_lock` sites are in `reasons/llm.py`. The lock itself is defined at `reasons/llm.py:57` as `_cost_lock = threading.Lock()`, which is a non-reentrant lock.

| Function | Lock site | What happens while holding `_cost_lock` | Calls another lock-acquiring function? |
|---|---:|---|---|
| `reset_cost_tracker` | `reasons/llm.py:71` | Resets `_cost_tracker` fields to zero/empty values | No |
| `get_cost_summary` | `reasons/llm.py:83` | Calls `copy.deepcopy(_cost_tracker)` and returns it | No; deepcopy operates on plain tracker data, not on locked functions |
| `format_cost_summary` | `reasons/llm.py:89` | Reads `_cost_tracker` and builds a formatted string | No |
| `_record_cost` | `reasons/llm.py:105` | Increments total and per-model call/token/cost counters | No |
| `invoke_model` | `reasons/llm.py:323` | In `finally`, adds elapsed seconds to total and per-model tracker entries | No; the locked block only updates dict values |

There is no locked path that calls another locked path. `_parse_cli_json` and `_invoke_openai` do call `_record_cost`, which acquires `_cost_lock`, but those calls happen from `invoke_model`’s normal `try` body before its `finally` lock acquisition. The lock acquisitions are sequential, not nested. Because `_cost_lock` is a plain `threading.Lock()` and not an `RLock`, nested acquisition would deadlock, but the observed code does not do that.

Also note that `reasons_lib/llm.py` is only a re-export shim:
```python
from reasons.llm import *  # noqa: F401,F403
```
so it adds no additional lock sites or locked call paths.

## Topics to Explore

- [file] `reasons/llm.py` — Contains all `_cost_lock` sites and the cost tracker implementation.
- [function] `reasons/llm.py:_record_cost` — Central lock-protected mutation path used by CLI, API, and OpenAI callers.
- [function] `reasons/llm.py:invoke_model` — Orchestrates model invocation and records elapsed time under `_cost_lock` in `finally`.
- [general] `cost-tracker-locking` — Explore whether future changes should use `threading.RLock` or keep strict non-reentrant lock discipline.
- [file] `reasons_lib/llm.py` — Confirms the public wrapper re-exports `reasons.llm` without additional locking behavior.

## Beliefs

- `cost-lock-is-non-reentrant` — `_cost_lock` is defined as `threading.Lock()` at `reasons/llm.py:57`, so nested acquisition would deadlock.
- `all-with-cost-lock-sites-are-in-reasons-llm` — The five `with _cost_lock` sites are all in `reasons/llm.py` at lines 71, 83, 89, 105, and 323.
- `record-cost-acquires-cost-lock` — `_record_cost` acquires `_cost_lock` to update global and per-model counters.
- `invoke-model-locks-only-in-finally` — `invoke_model` acquires `_cost_lock` only in its `finally` block while adding elapsed time.
- `locked-functions-do-not-reenter-cost-lock` — No function holding `_cost_lock` calls another function that acquires `_cost_lock`.
