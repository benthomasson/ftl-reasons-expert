---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_derive_budget.py

# `tests/test_derive_budget.py`

## 1. Purpose

This file is a **regression test suite** targeting a specific bug (issue #23) in `_build_beliefs_section` (in `reasons/derive.py`). The bug was that `count += len(belief_ids)` sat *inside* a per-belief loop, so with N agent beliefs the accumulator reached N×N instead of N. That inflated `count`, which in turn shrank the remaining budget for local beliefs via `remaining = max(5, max_beliefs - count)`, effectively starving the local-beliefs section.

The file's single responsibility: **lock in the correct linear budget-allocation arithmetic** across a matrix of agent counts, local-belief counts, budget sizes, and sampling modes, so the N² regression can never silently re-enter.

## 2. Key Components

### Test-Data Helpers (module-level functions)

| Function | Contract |
|----------|----------|
| `_make_nodes(agent_beliefs, local_beliefs=None)` | Returns `(nodes, derived, agents)` tuples shaped exactly as `_build_beliefs_section` expects. Builds **only** belief nodes (no `:active` premise nodes) to avoid inflating agent counts via `startswith` matching. `derived` is always `{}` unless a test overrides it. |
| `_count_local_shown(output)` | Regex-extracts the integer from `"Local beliefs (N beliefs, showing X)"`. **Raises `AssertionError`** if the header is absent. |
| `_count_agent_shown(output, agent_name)` | Same pattern for `"Agent: <name> (N beliefs, showing X)"`. |

### Fixtures

- **`db(tmp_path)`** — Creates an isolated SQLite DB via `api.init_db`. Only used by the one integration test (`TestBuildPromptIntegration`).

### Test Classes (grouped by concern)

| Class | What it pins down |
|-------|-------------------|
| `TestCountLinearNotQuadratic` | The core regression: with 6 agent beliefs and budget 20, locals must get 15 (not the buggy 5). Also verifies total ≤ budget with 10 agent / 50 local. |
| `TestMultiAgent` | Count **sums** across agents. Two and three-agent scenarios verify `local_shown == max(5, budget - Σ agent_shown)`. |
| `TestBudgetFloor` | The `max(5, …)` floor: even 50 or 200 agent beliefs can't push locals below 5. |
| `TestEdgeCases` | N=1 (where N²=N so the bug was invisible), no agents, no locals, empty network, agents-dict-with-no-matching-nodes, derived-belief exclusion. |
| `TestSampling` | `sample=True` still respects the budget floor; same `seed` → identical output (determinism). |
| `TestBuildPromptIntegration` | End-to-end through `build_prompt` with a real DB, verifying `stats["agents"]` and the local-belief budget. |

## 3. Patterns

- **Black-box / output-parsing testing.** The tests never reach into `_build_beliefs_section`'s internal variables. Instead they call it, get back a formatted string, and regex-extract the "showing N" counts. This means the tests are coupled to the **output format**, not the internal algorithm — a deliberate choice that catches user-visible regressions.
- **Regression-first test design.** The docstring at the top and each test's docstring explicitly state what the old buggy value *would* have been (e.g., "With the old bug: count=5*5=25, remaining=max(5, 20-25)=5"), making the test self-documenting.
- **Helper encapsulation of test-data construction.** `_make_nodes` isolates the "build a plausible `nodes`/`agents` dict" concern so individual tests read as pure assertions.
- **`tmp_path` fixture for DB isolation.** The single integration test gets a throwaway SQLite file; no shared state between tests.
- **Assert-as-oracle.** The helpers (`_count_local_shown`, `_count_agent_shown`) assert the regex matched before returning, so a format change in `derive.py` fails loudly in the helper rather than silently returning `None` and producing a confusing `int(None)` error.

## 4. Dependencies

**Imports (this file →):**

- `re` — regex parsing of the formatted output.
- `pytest` — fixture mechanism (`tmp_path`).
- `reasons.api` — `init_db`, `add_node`, `export_network` (integration test only).
- `reasons.derive` — `_build_beliefs_section` (the SUT) and `build_prompt` (integration).

**Imported by (→ this file):**

- Nothing. This is a leaf test module. It is discovered by pytest via the `tests/` directory convention.

## 5. Flow

```
┌─────────────────────────────────────────────────────────────┐
│  Unit tests (no DB)                                         │
│                                                             │
│  _make_nodes(...) ──► (nodes, derived, agents)              │
│                              │                              │
│                              ▼                              │
│         _build_beliefs_section(nodes, derived, agents,      │
│                                max_beliefs=N [, sample,seed])│
│                              │                              │
│                              ▼                              │
│                    formatted string `output`                │
│                              │                              │
│              ┌───────────────┴───────────────┐              │
│              ▼                               ▼              │
│   _count_local_shown(output)      _count_agent_shown(...)  │
│   (regex "showing N")            (regex "showing N")       │
│              │                               │              │
│              └───────────┬───────────────────┘              │
│                          ▼                                  │
│                 assert on budget arithmetic                  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Integration test (real SQLite)                             │
│                                                             │
│  db fixture ──► api.init_db(tmp_path)                       │
│       │                                                     │
│       ▼                                                     │
│  api.add_node(...) × (1 active + 6 agent + 20 local)        │
│       │                                                     │
│       ▼                                                     │
│  api.export_network(db) ──► data["nodes"]                   │
│       │                                                     │
│       ▼                                                     │
│  build_prompt(data["nodes"], budget=20)                     │
│       │                                                     │
│       ▼                                                     │
│  stats["agents"], _count_local_shown(prompt)                │
└─────────────────────────────────────────────────────────────┘
```

The unit tests bypass the database entirely — they hand-craft the `nodes`/`agents` dicts that `_build_beliefs_section` would normally read from the DB. The integration test exercises the full path: DB → `export_network` → `build_prompt` → `_build_beliefs_section`.

## 6. Invariants

These are the rules the tests **enforce** on `_build_beliefs_section`:

1. **Linearity of count.** With N total agent beliefs, the accumulated `count` must be exactly N (not N²). Verified in `test_six_beliefs_one_agent` (N=6 → locals get 15, not 5) and `test_count_equals_n_not_n_squared` (N=10 → total ≤ 20).

2. **Budget floor.** `local_shown ≥ 5` always, even when agents consume the entire budget (`test_floor_of_five_when_agents_exceed_budget`, `test_floor_applies_with_large_agent_set`).

3. **Budget ceiling.** `agent_shown + local_shown ≤ max_beliefs` (`test_count_equals_n_not_n_squared`, `test_build_prompt_budget_correct`).

4. **Multi-agent summation.** The count accumulates *across* all agents before computing the local remainder (`test_two_agents_accumulate_correctly`, `test_three_agents_budget_sums`).

5. **Determinism under sampling.** Same `seed` → byte-identical output (`test_sample_mode_deterministic`).

6. **Graceful emptiness.** Zero beliefs, zero agents, zero locals, or agents-dict with no matching nodes must not raise (`test_empty_network`, `test_empty_network_with_agents`, `test_no_agents`).

## 7. Error Handling

- **Helper assertion as guard.** `_count_local_shown` and `_count_agent_shown` call `assert m is not None, f"…"` immediately after `re.search`. If `_build_beliefs_section` ever changes its output format (e.g., renames "Local beliefs" to "Local Beliefs"), the test fails **in the helper** with a 500-char snippet of the actual output, rather than producing a cryptic `TypeError: int() argument must be a string, not 'NoneType'` two lines later.

- **No `try/except` anywhere in the file.** The tests let exceptions propagate naturally. A crash in `_build_beliefs_section` (e.g., a `KeyError` on a missing node) will surface as a standard pytest failure with a full traceback.

- **The `db` fixture** relies on pytest's `tmp_path` auto-cleanup — if `api.init_db` raises, the test is marked failed and the temp directory is discarded. No manual teardown.

- **No mocking or patching.** The tests call the real `_build_beliefs_section` with hand-built dicts. If the function signature changes, the tests fail at call-time with a clear `TypeError`.

---

## Topics to Explore

- [function] `reasons/derive.py:_build_beliefs_section` — The function under test. Understanding its internal loop structure, the `agent_budget` proportional split, and the `max(5, …)` floor is essential to interpreting every assertion in this file.
- [function] `reasons/derive.py:build_prompt` — The integration test calls this to exercise the full pipeline (DB export → prompt assembly). Tracing how it delegates to `_build_beliefs_section` and how `stats` is populated clarifies the boundary between unit and integration coverage.
- [file] `reasons/api.py` — `init_db`, `add_node`, and `export_network` are the DB-layer functions used in the integration test. Understanding the node schema (especially the `sl` "source link" and `label` fields) explains why `_make_nodes` must avoid `:active` premise nodes.
- [repo] `tests/test_derive.py` — The sibling test file for the broader derive module. Comparing its patterns against this budget-focused file reveals what `test_derive_budget.py` deliberately scoped out.
- [general] `proportional-budget-allocation` — The formula `agent_budget = max(5, int(budget * agent_count / total_count))` is a design choice worth understanding in context: how does it interact with multi-agent networks where one agent dominates?

## Beliefs

- `count-accumulation-is-linear-not-quadratic` — In `_build_beliefs_section`, the `count` variable must accumulate to exactly N (the total number of agent beliefs shown), never N²; the regression that made it N² is what this entire file prevents.
- `local-beliefs-floor-is-five` — Regardless of how many agent beliefs consume the budget, `_build_beliefs_section` guarantees `local_shown >= 5` via `remaining = max(5, max_beliefs - count)`.
- `_make-nodes-omits-active-premises` — The `_make_nodes` helper deliberately does **not** create `:active` premise nodes because `_build_beliefs_section` matches agent beliefs by `startswith(agent_name)`, and active-premise nodes would inflate the agent count and corrupt the budget arithmetic under test.
- `test-output-coupled-to-format-string` — The tests are black-box but fragile: they regex-match `"Local beliefs (\d+ beliefs, showing (\d+))"` and `"Agent: <name> (\d+ beliefs, showing (\d+))"`. Renaming or reformatting these headers in `derive.py` will break all 15+ tests here.
- `sample-mode-determinism-via-seed` — When `sample=True`, passing the same `seed` value must produce byte-identical output; this is asserted in `TestSampling::test_sample_mode_deterministic` and implies internal use of a seeded RNG rather than `random.sample`.
