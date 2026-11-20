---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Trace the original `count += len(belief_ids)` bug and why it produced quadratic behavior and local-budget starvation.

# Tracing the `count += len(belief_ids)` Bug

## Where the bug lived

The bug lived in `reasons/derive.py` inside `_build_beliefs_section`, in the branch that runs when `agents` are present (the multi-agent path, not the `intra_cluster`/`cluster` early-return paths, and not the `else` "Group by prefix" path). The offending line is still visible at **`reasons/derive.py:299`**, now correctly placed *outside* the per-belief loop:

```python
count = 0
for agent_name in sorted(agents, key=lambda a: -len(agents[a])):
    agent_beliefs = {k: v for k, v in in_nodes.items()
                    if k.startswith(f"{agent_name}:")}
    if not agent_beliefs:
        continue

    agent_budget = max(5, int(max_beliefs * len(agent_beliefs) / total_all))

    belief_ids = sorted(agent_beliefs.keys())
    if sample:
        belief_ids = _sample_beliefs(belief_ids, agent_budget, rng)
        belief_ids.sort()
    else:
        belief_ids = belief_ids[:agent_budget]

    lines.append(f"\n### Agent: {agent_name} (... showing {len(belief_ids)})")
    for belief_id in belief_ids:
        text = agent_beliefs[belief_id]["text"][:120]
        lines.append(f"- `{belief_id}`: {text}")
    count += len(belief_ids)          # <-- line 299, now after the loop
```

The pre-fix version had that last line **inside** the `for belief_id in belief_ids:` loop (per `tests/test_derive_budget.py:3`):

```python
for belief_id in belief_ids:
    text = agent_beliefs[belief_id]["text"][:120]
    lines.append(f"- `{belief_id}`: {text}")
    count += len(belief_ids)          # BUG: ran N times, each adding N
```

Because the loop iterated `belief_ids` (a list of length `N_i` for the current agent), the statement `count += len(belief_ids)` executed `N_i` times, adding `N_i` each time — `N_i²` contributed per agent.

## Why it was quadratic

Let `N_i = len(belief_ids)` after the per-agent capping/`sampling` step.

- **Intended semantics:** `count` tracks how many belief lines have been emitted so far from the agent section. After processing all agents, `count = Σ_i N_i` — linear in the number of displayed beliefs, and bounded by the sum of the per-agent budgets (roughly ≤ `max_beliefs`, modulo the `max(5, ...)` floor).
- **With the bug:** inside the per-agent loop, `count += len(belief_ids)` ran `N_i` times, so per-agent it added `N_i · N_i = N_i²`. Total across all agents is:
  ```
  count = Σ_i N_i²          (quadratic per agent)
  ```
  If one agent holds essentially the whole budget (`N_i ≈ max_beliefs`), then `count ≈ max_beliefs²`. With `max_beliefs = 300`, `count` can reach ~90,000 while only 300 beliefs were actually printed. The metric becomes quadratic in `N` while the true output size is linear.

Note this is an **accounting** quadratic, not a runtime one — the loop iterates exactly once per emitted belief line in either version. The quadratic blowup is entirely in the derived `count` value, which is then used downstream as a budget bookkeeping variable. Both the `sample=True` and `sample=False` branches of the agent path share this code, so the bug triggered in either mode.

## Why it starved the local budget

Immediately after the agent loop, the local (non-agent) beliefs get the leftover budget (`reasons/derive.py` around the `if non_agent:` block):

```python
if non_agent:
    remaining = max(5, max_beliefs - count)
    local_ids = sorted(non_agent.keys())
    if sample:
        local_ids = _sample_beliefs(local_ids, remaining, rng)
        local_ids.sort()
    else:
        local_ids = local_ids[:remaining]
```

So `remaining = max(5, max_beliefs - count)`.

With the correct count (`Σ N_i`), `remaining` is the true leftover — e.g., `20 - 5 = 15` in the canonical test case. With the buggy count (`Σ N_i²`), `count` aggressively overshoots `max_beliefs`, so `max_beliefs - count` is deeply negative and the `max(5, …)` floor **clamps every run to exactly 5 local beliefs**. Local (non-agent) beliefs are thereby starved to the floor regardless of how much of the actual budget remains — even in cases where the agent section consumed only, say, half the budget.

Concrete example from `tests/test_derive_budget.py:79-84` (`test_six_beliefs_one_agent`):
- 6 agent beliefs + 20 local beliefs, `max_beliefs=20`.
- `agent_budget = max(5, int(20 * 6 / 26)) = max(5, 4) = 5`, so `N_agent = 5`.
- **Correct:** `count = 5`, `remaining = max(5, 20 - 5) = 15` → locals show 15.
- **Buggy:** `count = 5 × 5 = 25`, `remaining = max(5, 20 - 25) = 5` → locals show 5.

## Why the bug was easy to miss

- **`N = 1` is silent:** `N² = N = 1`. The `test_single_agent_belief` test calls this out directly (`tests/test_derive_budget.py:176-187`). A network with one imported premise looked fine.
- **Multi-agent masking:** with many small agents (`N_i` small per agent), `Σ N_i²` may still be less than `max_beliefs`, so `remaining` doesn't collapse and locals still get budget. Starvation only manifested once a single agent dominated the network (or `N_i` grew enough that `N_i² > max_beliefs - 5`), which is exactly the pattern the tests exercise (`test_floor_of_five_when_agents_exceed_budget`, `test_floor_applies_with_large_agent_set`).
- **Proportional allocation hides it further:** each agent has a soft cap of `agent_budget_i ≈ max_beliefs · N_i / total_all`, so with many agents `N_i` stays small and `N_i²` stays small. The failure mode is a *large single agent* plus a meaningful local belief set — precisely the "import-agent then derive" flow.

## The fix

The fix is placement, not semantics: `count += len(belief_ids)` was moved from inside the per-belief loop to after it. Now each agent iteration adds `N_i` exactly once, so `count = Σ N_i` — equal to the number of belief lines actually appended to `lines`, and bounded appropriately by `max_beliefs`. The subsequent `remaining = max(5, max_beliefs - count)` therefore reflects the true leftover, letting local beliefs receive their full proportional share instead of hitting the floor of 5. The regression test file `tests/test_derive_budget.py` exists specifically to pin this behaviour (its docstring is the one that appears in the grep at line 3).

## Topics to Explore

- [file] `reasons/derive.py` — Full `_build_beliefs_section` including the `cluster` / `intra_cluster` early-return paths that bypass the agent loop; verify whether similar accounting exists there.
- [function] `reasons/derive.py:_sample_beliefs` — Reservoir sampling used in both agent and local branches; important for understanding how `len(belief_ids)` is decided before `count` accumulation.
- [file] `tests/test_derive_budget.py` — Regression tests encoding the expected `count = Σ N_i` invariant, including the multi-agent accumulation and budget-floor cases.
- [general] `proportional-budget-allocation` — The `max(5, int(max_beliefs * len(agent_beliefs) / total_all))` formula; explore how the `max(5, …)` floor interacts with the post-loop `count` to produce the overall `max_beliefs` cap (and where it can still overflow in pathological multi-agent networks).
- [diff] `299` — Compare the current line to the pre-fix placement; understand that the change is purely loop-scoping, not a semantic change to `count`'s meaning.

## Beliefs

- `count-accumulates-agent-beliefs-linearly` — In the fixed code, `count` in `_build_beliefs_section` equals `Σ_i len(belief_ids_i)` after the agent loop, matching the total number of belief lines emitted from the agent section.
- `count-bug-produces-quadratic-metric` — The original buggy placement (`count += len(belief_ids)` inside the per-belief loop) made `count = Σ_i N_i²`, quadratic in per-agent belief counts, instead of the true `Σ_i N_i`.
- `count-feeds-local-budget` — The remaining budget for non-agent beliefs is computed as `remaining = max(5, max_beliefs - count)`, so any inflation of `count` directly starves the local section down to its floor of 5.
- `count-bug-causes-local-budget-starvation` — Because inflated `count` exceeds `max_beliefs`, the `max(5, …)` floor clamps local beliefs to exactly 5 whenever `Σ_i N_i² ≥ max_beliefs - 5`.
- `count-bug-is-invisible-at-n-equals-one` — When a given agent contributes exactly one belief, `N_i² = N_i`, so the bug does not alter `count` for that agent and the regression is only observable with `N_i ≥ 2`.
