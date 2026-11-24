---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Investigates cases where quota caps cause fewer than `budget` beliefs to be selected.

There is no literal `quota` symbol in the observed codebase; the quota behavior is implemented conceptually in `reasons/derive.py` by two caps:

- per-agent proportional budget in `_build_beliefs_section`
- local-belief remaining budget

`build_prompt` passes the caller’s `budget` directly as `max_beliefs` into `_build_beliefs_section` (`reasons/derive.py:531-537`), so quota under-selection happens inside that function.

## Where the quota logic lives

`_build_beliefs_section` is at `reasons/derive.py:185`. The relevant agent branch starts at `reasons/derive.py:266`.

For each agent it computes:

```python
agent_budget = max(5, int(max_beliefs * len(agent_beliefs) / total_all))
```

at `reasons/derive.py:284`, then truncates or samples the agent’s beliefs to that cap (`reasons/derive.py:288-293`). After the per-agent loop it accumulates the actual selected count:

```python
count += len(belief_ids)
```

at `reasons/derive.py:300`.

For local, non-agent beliefs it computes:

```python
remaining = max(5, max_beliefs - count)
```

at `reasons/derive.py:304`, then truncates/samples locals to `remaining` (`reasons/derive.py:305-310`).

The old bug referenced in `tests/test_derive_budget.py` was that `count += len(belief_ids)` was inside the per-belief loop, inflating `count` to N² and starving locals through `remaining = max(5, max_beliefs - count)`. The current code increments after the loop.

## When quota caps select fewer than `budget`

### 1. Natural under-supply

If the total eligible `IN` non-derived beliefs are fewer than `budget`, the function cannot select `budget`. This is not a quota bug.

Examples in `tests/test_derive_budget.py`:

- `test_single_agent_belief` at `tests/test_derive_budget.py:170` has 1 agent belief + 10 locals, budget 20.
- `test_no_agents` at `tests/test_derive_budget.py:182` has 10 beliefs, budget 20.

### 2. Per-agent proportional truncation leaves unallocated budget

The per-agent cap uses:

```python
max(5, int(max_beliefs * len(agent_beliefs) / total_all))
```

Because of `int(...)`, the sum of per-agent budgets can be less than `max_beliefs`. The code does not redistribute unused per-agent quota to other agents. Local beliefs can consume the leftover via `remaining`, but only if local beliefs exist and only up to `remaining`.

Concrete under-selection case:

- `max_beliefs = 20`
- three agents, each with 7 beliefs
- no local beliefs

Then:

```text
total_all = 21
agent_budget = max(5, int(20 * 7 / 21)) = max(5, 6) = 6
```

Each agent shows 6 beliefs, so total selected is:

```text
6 + 6 + 6 = 18
```

Since `non_agent` is empty, there is no local section to fill the remaining 2 slots. The result is 18 selected beliefs even though `budget=20`.

If there is 1 local belief, `remaining = max(5, 20 - 18) = 5`, but only 1 local is available, so total selected is 19. If there are 2 locals, total selected reaches 20. If there are enough locals, the floor can even push the total above budget.

### 3. Local remaining cap does not always fill the gap

`remaining = max(5, max_beliefs - count)` at `reasons/derive.py:304` guarantees locals can get at least 5 slots when a local section exists. But it does not guarantee the overall total reaches `budget`:

- if `count` is already near budget, `remaining` can be 5 and overshoot
- if there are fewer local beliefs than `remaining`, the gap remains
- unused per-agent quota is not reallocated to other agents

### 4. Clustered selection may also under-select

When `cluster=True` or `intra_cluster=True`, `_build_beliefs_section` delegates selection to `cluster_beliefs` or `cluster_beliefs_intra` (`reasons/derive.py:211-225`). If those functions return fewer than `max_beliefs`, the agent branch does not top up the selection. The observations do not include `reasons/cluster.py`, so that behavior is not fully visible here.

## What the tests cover

`tests/test_derive_budget.py` is focused on the N²/starving-locals regression:

- `test_six_beliefs_one_agent` at `tests/test_derive_budget.py:65` verifies locals get 15 instead of the buggy 5.
- `test_count_equals_n_not_n_squared` at `tests/test_derive_budget.py:81` verifies `agent_shown + local_shown <= 20`.
- `test_two_agents_accumulate_correctly` at `tests/test_derive_budget.py:98` verifies local budget equals `max(5, 20 - agent_total)`.
- `test_no_local_beliefs` at `tests/test_derive_budget.py:191` covers an agent-only network, but only with 5 agent beliefs and budget 20, so it is already an under-supply case.

The tests do not appear to include the specific multi-agent fractional-quota case above, where enough beliefs exist overall but per-agent truncation plus insufficient locals causes fewer than `budget` beliefs to be selected.

## Topics to Explore

- [file] `reasons/derive.py` — Inspect `_build_beliefs_section` lines 185–349 and `build_prompt` lines 439–591.
- [function] `reasons/derive.py:_build_beliefs_section` — Focus on agent quota allocation and local `remaining` calculation.
- [file] `reasons/cluster.py` — Check whether `cluster_beliefs` and `cluster_beliefs_intra` can also return fewer than `max_beliefs`.
- [file] `tests/test_derive_budget.py` — Add a regression test for multi-agent fractional quotas with no or too few local beliefs.
- [general] `quota-redistribution` — Decide whether unused per-agent quota should be redistributed to other agents to fill `budget`.

## Beliefs

- `agent-quota-uses-proportional-budget` — Each agent gets `max(5, int(max_beliefs * agent_count / total_all))` in `_build_beliefs_section`.
- `count-is-incremented-after-agent-loop` — `count += len(belief_ids)` is outside the per-belief loop at `reasons/derive.py:300`, fixing the N² count inflation.
- `local-budget-floors-at-five` — `remaining = max(5, max_beliefs - count)` gives locals at least 5 slots whenever a local section exists.
- `unused-agent-quota-is-not-redistributed` — Leftover per-agent quota does not flow back to other agents; only `non_agent` locals can consume `remaining`.
- `quota-truncation-can-underfill-budget` — Integer truncation in per-agent budgets can make total selected beliefs less than `max_beliefs` when locals are absent or fewer than `remaining`.
