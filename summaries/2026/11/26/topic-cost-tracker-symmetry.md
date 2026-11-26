---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Cross-backend invariant: which counters should exist for every backend regardless of whether it reports usage, and how to enforce that in code rather than by convention.

From the observations, this invariant is about **LLM backend usage counters**, not SQLite/PostgreSQL storage counters. The code has two different LLM response-parsing paths, and they currently normalize usage by convention rather than through a shared contract.

## What the invariant should be

Every LLM backend should return a normalized result shape containing usage counters even when the provider does not report usage:

- `usage.input_tokens` — always present, integer, defaulting to `0`
- `usage.output_tokens` — always present, integer, defaulting to `0`
- `total_cost_usd` — always present, numeric, defaulting to `0.0`

If the backend exposes cache token details, they should be preserved if useful, but they must be consistently folded into `input_tokens` so downstream callers do not have to specialize per backend.

The evidence is in `reasons/llm.py`:

- Lines 149–151 parse a CLI/JSON-style response:
  ```python
  usage = data.get("usage", {})
  input_tokens = usage.get("input_tokens", 0) + usage.get("cache_creation_input_tokens", 0) + usage.get("cache_read_input_tokens", 0)
  output_tokens = usage.get("output_tokens", 0)
  ```
  This shows the intended normalization: `input_tokens` includes cache creation/read tokens, and missing values default to `0`.

- Lines 269–270 parse an SDK-style response:
  ```python
  input_tokens = response.usage.input_tokens if response.usage else 0
  output_tokens = response.usage.output_tokens if response.usage else 0
  ```
  This path also defaults missing usage to `0`, but it does **not** include cache creation/read tokens. That is exactly the kind of cross-backend drift an explicit invariant should prevent.

`tests/test_llm.py` already has examples of empty usage:
- Lines 67, 121, 129, 172: mocks return `"usage": {}` and `"total_cost_usd": 0.0`
- Line 342: explicit `{"input_tokens": 100, "output_tokens": 50}`

So the tests encode some expected behavior, but there is no single enforcement point that guarantees all backends produce the same normalized counters.

## How to enforce it in code instead of convention

The observations do not show the full backend class/protocol around `reasons/llm.py`, so I cannot name the exact backend classes. But the enforcement point should be the shared boundary where both LLM response paths produce their final result.

Recommended enforcement:

1. **Define a single normalized result type**  
   Use a dataclass or `TypedDict` with required fields, e.g.:
   ```python
   @dataclass(frozen=True)
   class Usage:
       input_tokens: int = 0
       output_tokens: int = 0
       cache_creation_input_tokens: int = 0
       cache_read_input_tokens: int = 0

   @dataclass(frozen=True)
   class LLMResult:
       result: str
       usage: Usage
       total_cost_usd: float = 0.0
   ```

2. **Centralize usage extraction**  
   Replace the duplicated logic at `reasons/llm.py:149-151` and `reasons/llm.py:269-270` with one helper:
   ```python
   def normalize_usage(raw_usage: Mapping[str, Any] | Any | None) -> dict:
       def val(name: str) -> int:
           if raw_usage is None:
               return 0
           if isinstance(raw_usage, Mapping):
               return int(raw_usage.get(name, 0) or 0)
           return int(getattr(raw_usage, name, 0) or 0)

       cache_create = val("cache_creation_input_tokens")
       cache_read = val("cache_read_input_tokens")
       return {
           "input_tokens": val("input_tokens") + cache_create + cache_read,
           "output_tokens": val("output_tokens"),
           "cache_creation_input_tokens": cache_create,
           "cache_read_input_tokens": cache_read,
       }
   ```

   Then every backend calls this helper, whether the raw usage is a JSON dict or an SDK object.

3. **Normalize the outer result too**  
   Ensure every backend returns `result`, `usage`, and `total_cost_usd`. A factory like `normalize_llm_result(...)` should own the defaulting:
   ```python
   def normalize_llm_result(result: str, usage_raw=None, total_cost_usd=0.0) -> dict:
       return {
           "result": result,
           "usage": normalize_usage(usage_raw),
           "total_cost_usd": float(total_cost_usd or 0.0),
       }
   ```

4. **Make tests parameterized across backends**  
   Add tests that run every backend mode through the same assertions:
   - missing usage returns `input_tokens == 0`, `output_tokens == 0`, `total_cost_usd == 0.0`
   - cache usage is folded into `input_tokens`
   - SDK `usage=None` returns zeros
   - all returned results contain the same required keys

   This is stronger than the current per-backend tests in `tests/test_llm.py`, which do not prove cross-backend equivalence.

5. **Use the existing metadata pattern as a model**  
   `reasons/metadata.py:build_meta` is already a single source of truth for metadata fields and defaults. The usage counters need the same kind of centralized builder/normalizer rather than scattered `.get(..., 0)` calls.

The key architectural rule is: **backend-specific code should not be allowed to return a raw provider response**. It must pass through the shared normalizer, and that normalizer must be covered by cross-backend tests.

## Topics to Explore

- [file] `reasons/llm.py` — Inspect both usage extraction paths at lines 149–151 and 269–270; these are the current divergence points.
- [file] `tests/test_llm.py` — Review existing tests for empty usage, explicit usage, and cache-token behavior.
- [file] `reasons/metadata.py` — See how `build_meta` centralizes field names and defaults as a model for a usage normalizer.
- [general] `llm-backend-protocol` — Identify the exact backend interface/classes in `reasons/llm.py` that should return a normalized `LLMResult`.
- [general] `usage-counter-normalization` — Design the shared helper and parameterized tests that enforce `input_tokens`, `output_tokens`, and `total_cost_usd` across all backends.

## Beliefs

- `cli-json-usage-defaults-to-zero` — The CLI/JSON LLM path uses `data.get("usage", {})` and `.get(..., 0)` for token counters at `reasons/llm.py:149-151`.
- `cli-input-tokens-includes-cache-tokens` — At `reasons/llm.py:150`, normalized input tokens add `input_tokens`, `cache_creation_input_tokens`, and `cache_read_input_tokens`.
- `sdk-usage-defaults-to-zero` — At `reasons/llm.py:269-270`, SDK usage counters default to `0` when `response.usage` is `None`.
- `sdk-input-tokens-does-not-include-cache-tokens` — At `reasons/llm.py:269`, SDK input tokens are read only from `response.usage.input_tokens`, unlike the CLI path.
- `llm-tests-accept-empty-usage` — `tests/test_llm.py` CLI mocks return `"usage": {}` with `"total_cost_usd": 0.0` at lines 67, 121, 129, and 172.
