---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: _sample_beliefs in reasons/derive.py

## Purpose

`_sample_beliefs` is a private helper used to cap the number of belief IDs shown to the LLM when prompt construction is in sampling mode. If there are more beliefs than the configured budget, it returns a random subset of exactly `budget` elements; otherwise it returns all belief IDs unchanged.

Its practical role is to avoid always truncating alphabetically or by insertion order, which would bias the prompt toward a subset of the belief network. It is used by `_build_beliefs_section` when `sample=True`.

There is an important documentation mismatch: the docstring says “Uses reservoir sampling,” but the implementation delegates to `rng.sample(...)`, which is Python’s built-in random sampling-without-replacement routine, not reservoir sampling.

## Contract

Preconditions:

- `belief_ids` should be a sized sequence, usually a `list`.
- When sampling occurs, it must also be accepted by `random.Random.sample`, which expects an indexable sequence.
- `budget` should be a non-negative integer.
- `rng`, if supplied, must provide a `sample(population, k)` method compatible with `random.Random.sample`.

Postconditions:

- If `len(belief_ids) <= budget`, the original object is returned unchanged. No copy is made, no shuffle occurs, and no randomness is consumed.
- If `len(belief_ids) > budget`, a new list containing exactly `budget` elements is returned.
- The returned elements are selected without replacement.
- If a seeded `rng` is supplied, sampling is reproducible according to that RNG’s state.

Invariants/assumptions not enforced by types:

- `belief_ids` items are intended to be unique belief IDs. If duplicates existed, sampling is by position, not by uniqueness of value.
- The caller generally expects the returned value to be sortable/mutable when the original input is a list, but the function itself does not normalize the return type.
- `budget` is assumed non-negative. A negative budget is not guarded.
- `rng.sample` assumes `budget` is an integer when sampling actually occurs.

## Parameters

- `belief_ids`
  - Typically a `list[str]` of belief IDs.
  - In one call site in `_build_beliefs_section`, it is a `list[tuple[str, str]]`.
  - Any sized object works for the early-return branch; the over-budget branch needs a sequence accepted by `random.sample`.

- `budget`
  - Maximum number of elements to return.
  - Expected to be a non-negative integer.
  - If `budget >= len(belief_ids)`, all elements are returned.
  - If `budget == 0` and `len(belief_ids) > 0`, sampling returns an empty list.
  - Negative values cause `random.sample` to raise `ValueError`.

- `rng`
  - Optional random source, usually a `random.Random` instance.
  - If provided, `_sample_beliefs` calls `rng.sample(...)`.
  - If omitted and sampling is needed, the function constructs a fresh `random.Random()`.
  - Providing an object without a compatible `sample` method raises `AttributeError`.

## Return Value

- If `len(belief_ids) <= budget`: returns `belief_ids` itself, preserving its type and identity.
- If `len(belief_ids) > budget`: returns a new `list` of sampled elements.
- The returned list from `rng.sample` is in random selection order, not sorted.
- The function never returns `None`.

Callers must handle:

- Possible aliasing when the input is returned unchanged.
- The fact that a short list is not shuffled.
- The fact that a sampled result may need sorting if the caller wants deterministic display order.

## Algorithm

1. Check `len(belief_ids) <= budget`.
2. If true, return the original `belief_ids` object immediately.
3. Otherwise, sampling is required.
4. If `rng is None`, create a new `random.Random()` instance.
5. Call `rng.sample(belief_ids, budget)`.
6. Return the resulting list.

So the actual behavior is:

```python
if len(belief_ids) <= budget:
    return belief_ids
if rng is None:
    rng = random.Random()
return rng.sample(belief_ids, budget)
```

## Side Effects

- If sampling occurs and no RNG is supplied, a new `random.Random()` is constructed, which may seed from system entropy or time. It does not use or alter the module-level `random` global state.
- If an RNG is supplied, its internal state advances when `.sample()` is called.
- `_sample_beliefs` does not mutate the input population.
- However, if it returns the input unchanged, the caller could mutate the same object afterward. In current call sites, the caller sometimes sorts the returned list, but the input is a freshly created `sorted(...)` list, so this is safe there.

## Error Handling

The function has no explicit error handling and does not swallow exceptions.

Possible exceptions:

- `TypeError` if `belief_ids` has no length or is not comparable to `budget`.
- `TypeError` if `budget` is not accepted by `random.sample` when sampling occurs, e.g. a float.
- `ValueError` if `budget` is negative and sampling occurs.
- `AttributeError` if a supplied `rng` lacks a `.sample()` method or if its `sample` contract differs.

Because the early return handles `len(belief_ids) <= budget`, the common “sample larger than population” case is avoided for ordinary integer budgets.

## Usage Patterns

`_sample_beliefs` is used inside `_build_beliefs_section` only in non-cluster sampling branches:

- For agent-specific beliefs:
  ```python
  belief_ids = sorted(agent_beliefs.keys())
  if sample:
      belief_ids = _sample_beliefs(belief_ids, agent_budget, rng)
      belief_ids.sort()
  ```

- For local/non-agent beliefs:
  ```python
  local_ids = sorted(non_agent.keys())
  if sample:
      local_ids = _sample_beliefs(local_ids, remaining, rng)
      local_ids.sort()
  ```

- For the non-agent grouped path:
  ```python
  sampled_keys = set(
      k for k, _ in _sample_beliefs(all_items, max_beliefs, rng)
  )
  ```

The RNG is created once in `_build_beliefs_section` as `random.Random(seed)` when `sample=True`, then reused across successive sampling calls. That means reproducibility depends on call order and how many previous samples consumed RNG state.

## Dependencies

- Standard library `random`.
- No internal module dependencies.
- Relies on `random.Random.sample`, which is part of Python’s standard `random` API.

## Topics to Explore

- [function] `reasons/derive.py:_build_beliefs_section` — Shows how `_sample_beliefs` is called, how budgets are allocated per agent, and how sampling interacts with clustering.
- [function] `reasons/derive.py:build_prompt` — Connects prompt construction, `sample`, `seed`, and budget configuration to the final LLM prompt.
- [file] `tests/test_derive_budget.py` — likely contains budget and sampling behavior tests relevant to this helper.
- [file] `tests/test_derive.py` — covers parsing, validation, and derivation behavior around the derive pipeline.
- [general] `random-sample-vs-reservoir-sampling` — Useful for understanding the docstring mismatch and the tradeoffs between `random.sample` and true reservoir sampling.

## Beliefs

- `_sample_beliefs-returns-input-unchanged-when-under-budget` — If `len(belief_ids) <= budget`, the function returns the original object without copying or shuffling it.
- `_sample_beliefs-uses-random-sample-not-reservoir-sampling` — Despite the docstring, the implementation calls `rng.sample` for over-budget sampling rather than reservoir sampling.
- `_sample_beliefs-requires-non-negative-budget` — A negative `budget` reaches `rng.sample` and raises `ValueError`.
- `_sample_beliefs-does-not-touch-global-random-state` — When `rng` is omitted and sampling occurs, it creates a new `random.Random()` instead of using module-level random functions.
- `_sample_beliefs-preserves-population` — The input sequence is not mutated; over-budget sampling returns a new list.
