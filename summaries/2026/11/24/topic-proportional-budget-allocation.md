---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: The `max(5, int(max_beliefs * len(agent_beliefs) / total_all))` formula; explore how the `max(5, …)` floor interacts with the post-loop `count` to produce the overall `max_beliefs` cap (and where it can still overflow in pathological multi-agent networks).

## What the formula computes

At `reasons/derive.py:283`:

```python
agent_budget = max(5, int(max_beliefs * len(agent_beliefs) / total_all))
```

This is a **per-agent proportional quota**:

- `total_all` is built at `reasons/derive.py:272` as `total_agent_beliefs + len(non_agent)`.
- `len(agent_beliefs)` is the number of candidate beliefs for the current agent.
- `len(agent_beliefs) / total_all` is that agent’s share of all candidate beliefs.
- Multiplying by `max_beliefs` gives the agent’s fair share of the global budget.
- `int(...)` floors the fair share.
- `max(5, ...)` guarantees every agent gets at least 5 slots.

So the formula means: “Give this agent a share of the global belief budget proportional to how many beliefs it owns, but never less than 5.”

## How the floor interacts with `count` and the global cap

The formula itself does **not** enforce the global `max_beliefs` cap. It only computes a per-agent budget. The actual global cap comes from the accumulated `count` compared against `max_beliefs` at `reasons/derive.py:339` and `reasons/derive.py:343`.

There is also a second floor at `reasons/derive.py:303`:

```python
remaining = max(5, max_beliefs - count)
```

That is important: when fewer than 5 global slots remain, `remaining` still returns `5`. So if code consumes `remaining` without an additional hard check, it can overshoot `max_beliefs` by up to 4 beliefs. If the code adds a whole per-agent quota or a chunk of beliefs after the cap is nearly reached, the overshoot can be larger.

The intended behavior is:

1. `agent_budget` limits how many beliefs are considered for that agent.
2. `count` tracks how many beliefs have actually been emitted.
3. The checks at lines `339` and `343` stop or skip further output once `count >= max_beliefs`.
4. If sampling is enabled, `_sample_beliefs(all_items, max_beliefs, rng)` at `reasons/derive.py:325` appears to apply another global cap.

So the overall cap is produced by the combination of the per-agent quota, the mutable `count`, the `count >= max_beliefs` checks, and optionally the sampling helper. The `max(5, ...)` floor makes the per-agent quotas intentionally biased toward minimum representation, which can conflict with the global cap.

## Where it can still overflow

The main pathological case is **many agents with small belief shares**.

Example: `max_beliefs = 20`, and there are 10 agents with 10 beliefs each.

- `total_all = 100`
- For each agent: `int(20 * 10 / 100) = int(2) = 2`
- Then `max(5, 2) = 5`
- Sum of per-agent quotas = `10 * 5 = 50`, which is more than twice `max_beliefs`.

The global cap only holds if `count` is checked frequently enough. If the checks at `reasons/derive.py:339` and `:343` happen before each append, the output can stop at 20 but later agents may be starved. If the check happens only after an entire agent’s beliefs are emitted, the first four agents can fill the 20-slot budget, and the fifth agent can push the total to 25. If the check happens only after the full multi-agent loop, the total could grow toward `5 * number_of_agents`.

Other overflow-prone cases:

- `max_beliefs < 5`: every agent still gets at least 5, so the per-agent floor alone exceeds the global cap.
- `count` is close to `max_beliefs`, e.g. `count = 18`, `max_beliefs = 20`; `remaining = max(5, 2) = 5`, so consuming `remaining` can produce `23`.
- A large `non_agent` list inflates `total_all`, shrinking all agent fair shares below 5 and triggering the floor for many agents.
- Sampling is disabled or `_sample_beliefs` is not the final operation, so the `count >= max_beliefs` checks are the only protection.

The exact worst-case bound cannot be determined from the supplied observations alone. The missing piece is the loop body between `reasons/derive.py:287` and `reasons/derive.py:343`, especially whether `count += 1` happens before or after the `count >= max_beliefs` checks and whether `remaining` is used as an unbounded slice count.

## Topics to Explore

- [function] `reasons/derive.py:_build_beliefs_section` — Inspect lines 267–343 to see the exact order of `count += 1`, `remaining`, and the `count >= max_beliefs` checks.
- [function] `reasons/derive.py:_sample_beliefs` — Confirm whether sampling hard-limits output to `max_beliefs` and how it interacts with per-agent budgets.
- [file] `tests/test_derive_budget.py` — Look at boundary tests using `max_beliefs=15`, `20`, and `30` to see expected multi-agent floor behavior.
- [file] `reasons/derive.py` — Review how `non_agent` is defined and whether non-agent beliefs share the same `count` budget.
- [general] `multi-agent-budget-pathology` — Construct worst-case many-agent sparse-belief networks and verify the maximum possible final `count`.

## Beliefs

- `max-beliefs-formula-is-per-agent-proportional-allocation` — `reasons/derive.py:283` computes a per-agent quota from the agent’s share of `total_all`, not a global cap.
- `floor-of-five-can-exceed-global-budget` — The `max(5, ...)` floor at `reasons/derive.py:283` can make the sum of per-agent quotas exceed `max_beliefs` when there are many agents.
- `global-cap-is-enforced-by-count-checks` — Lines `reasons/derive.py:339` and `:343` compare `count` to `max_beliefs`; the formula alone does not enforce the global cap.
- `remaining-floor-can-overshoot` — `reasons/derive.py:303` returns at least 5 even when fewer than 5 global slots remain, so using `remaining` without another bound can exceed `max_beliefs`.
- `sampling-path-reapplies-global-cap` — `reasons/derive.py:325` passes `max_beliefs` into `_sample_beliefs`, making the sampling path a second control point for the global cap.
