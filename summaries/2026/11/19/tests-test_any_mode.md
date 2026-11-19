---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_any_mode.py

# Explanation: `tests/test_any_mode.py`

## Purpose

This is a **contract-focused integration test suite** for three interrelated features of the `reasons` package, each tied to a GitHub issue:

| Issue | Feature | Where tested |
|-------|---------|--------------|
| **#20** | `--any` flag on `add`/`add-justification` expands a conjunctive SL into per-premise justifications (OR semantics instead of AND) | `TestAnyModeAdd`, `TestAnyModeAddJustification` |
| **#19** | `premise_count` in return values gives the CLI enough info to warn on 3+ premise support links | `TestPremiseCountWarning` |
| **#21** | `retraction_hints` on `retract_node` helps users understand why a node went OUT and what premises survive | `TestRestorationHints` |

The file deliberately exercises the **public `reasons.api` surface only** (`add_node`, `add_justification`, `retract_node`, `show_node`, `init_db`) — no internal helpers, no monkeypatching. It therefore doubles as executable specification for the API's contract, not just a regression suite.

---

## Key Components

### Fixture
```python
@pytest.fixture
def db(tmp_path):
    db_path = str(tmp_path / "reasons.db")
    api.init_db(db_path=db_path)
    return db_path
```
- **Function-scoped** (pytest default) → every test gets a fresh, isolated SQLite database under `tmp_path`.
- Returns a **path string**, not a connection — every API call in the tests passes `db_path=db` to open its own handle. This confirms the API is stateless w.r.t. connections and re-entrant per call.
- No teardown needed: pytest garbage-collects `tmp_path`.

### `TestAnyModeAdd` (#20)
Covers `api.add_node(..., any_mode=True)`. Behaviors verified:
- **`sl="a,b,c"` + `any_mode=True` → 3 justifications, each with 1 antecedent** (`test_any_creates_separate_justifications`). This is the core OR-vs-AND semantic: without `any_mode`, the same input creates **1** justification with **3** antecedents (`test_without_any_single_justification`).
- **Retraction resilience**: retracting one premise leaves `x` `"IN"` because two justifications remain (`test_any_survives_single_retraction`); retracting all premises flips to `"OUT"` (`test_any_goes_out_when_all_retracted`).
- **`unless` propagation**: each expanded justification carries the full outlist, so `"enemy" in j["outlist"]` holds for all expanded JSLs (`test_any_preserves_outlist`).
- **Degenerate case**: single-premise `sl="a"` produces 1 justification, not 1×(trivial) expansion (`test_any_with_single_premise_no_expansion`).

### `TestAnyModeAddJustification` (#20)
- **`test_any_expands_add_justification`**: adding an `any_mode` SL to a node that already has an `SL(a)` justification results in **3** justifications total (the original + b-only + c-only). Confirms expansion happens *per call*, appended alongside existing justifications, not merged into one.
- **`test_any_add_justification_restores_out_node`**: an OUT node can be brought back to IN by adding a fresh `any_mode` justification whose premises are IN. Return value includes `result["new_truth_value"] == "IN"`.

### `TestPremiseCountWarning` (#19)
Pins down the exact value of `result["premise_count"]` for every add path:
- `add_node` with N-premise SL → `N` (`test_add_returns_premise_count`, N=3).
- `add_node(any_mode=True)` with N-premise SL → **`1`**, not `N` (`test_add_any_returns_low_premise_count`). The count reflects **one** justification's antecedent count, which is what the CLI needs to avoid spurious warnings.
- `add_justification` with N premises → `N`.
- `add_node` with no `sl` (a premise) → `0` (`test_premise_returns_zero_count`).

### `TestRestorationHints` (#21)
Verifies the shape and gating of `retract_node(...)["restoration_hints"]`:
- Each hint is a dict with `node_id`, `all_premises`, `surviving_premises` (see `test_hint_when_multi_premise_cascades`).
- **Gate 1 — multi-premise only**: single-premise SLs do not hint (`test_no_hint_for_single_premise_sl`).
- **Gate 2 — surviving premises required**: if the cascade takes *all* premises OUT, no hint (`test_no_hint_when_all_premises_out`).
- **Gate 3 — not the retracted node itself**: `retract_node("a")` with no dependents → `restoration_hints == []` (`test_no_hint_for_directly_retracted_node`).
- **Gate 4 — any-mode nodes skipped**: because each `any_mode` justification has one antecedent, they never trigger the hint logic (`test_hint_with_any_flag_no_false_alarm`); the node also stays `"IN"`.
- **Cascade correctness**: `a` is retracted, `b` was `SL(a)` so cascades OUT, leaving only `c` in `surviving_premises` (`test_hint_includes_correct_surviving_premises`). Asserts exact list equality — this both checks membership and that ordering is deterministic (single-element list, but the contract is order-sensitive).

---

## Patterns

1. **Class-scoped feature grouping**: each issue gets its own `Test*` class with a docstring citing the issue number. New bugs → new test methods in the appropriate class.
2. **Fixture-based isolation via `tmp_path`** — no global DB, no shared state, no cleanup boilerplate.
3. **Black-box / contract testing**: all assertions go through `api.*` and inspect dict return values. Storage/derivation internals are never imported, so refactoring `reasons.storage` cannot break these tests unless the observable contract changes.
4. **Setup-act-assert triples** using the API itself — nodes are created via `add_node`, not direct SQL. This means a regression in `add_node` will surface in *these* tests *before* the actual feature under test, which is acceptable here because the tests are about the composed system.
5. **Negation tests as first-class cases** — every feature has at least one test asserting *absence* of behavior (`test_without_any_single_justification`, `test_no_hint_*`). This pins down that the feature is opt-in, not accidental.
6. **Return-value contracts are explicit key lookups** — `result["new_truth_value"]`, `result["premise_count"]`, `result["restoration_hints"]`. These are the CLI-facing fields, so any rename or removal is a test failure here.

---

## Dependencies

**Imports**
- `pytest` — test runner, fixtures (`tmp_path`), class-style test discovery.
- `from reasons import api` — the module under test. Note it imports the `api` submodule directly, not `import reasons` or `from reasons_lib import ...`. The repo has both `reasons/` and `reasons_lib/` trees; this suite pins the API surface to `reasons.api`.

**Depends on (indirectly, at runtime)**
- `reasons.storage` — persistence for nodes/justifications.
- `reasons` inference engine — for `truth_value` recomputation after retraction.
- pytest's `tmp_path` fixture (built-in).

**Depended on by**
- CI: the tests are collected by pytest under the standard `tests/` directory (see `.github/workflows`).
- Nothing imports this test module; it's a leaf.

---

## Flow (per test)

1. `init_db(db_path=...)` sets up schema in a fresh `tmp_path/reasons.db`.
2. A chain of `api.add_node("a", "A", db_path=db)` calls registers premises (and any intermediate SLs).
3. The feature under test is invoked:
   - `add_node(..., any_mode=True)` / `add_justification(..., any_mode=True)`, or
   - `retract_node(...)`.
4. Assertions read either:
   - the return value of the mutating call (`premise_count`, `new_truth_value`, `restoration_hints`), or
   - a follow-up `show_node(...)` snapshot (`justifications[*].antecedents`, `justifications[*].outlist`, `truth_value`).
5. Fresh `tmp_path` per test means steps 1–2 run from scratch; no inter-test ordering.

Data transformations:
- Input `sl="a,b,c"` → parsed into 3 antecedents → with `any_mode`, fanned out into 3 `Justification` records, each with `antecedents={"a"}`, `{"b"}`, `{"c"}` and a copied `outlist`.
- `retract_node("a")` → propagates `"OUT"` through dependents → for each node that flips OUT and has ≥2 premises and ≥1 surviving premise (and isn't the direct target), emit a hint dict.

---

## Invariants

- **Cardinality invariant**: `any_mode=True` guarantees *every resulting justification has exactly 1 antecedent*. The `for j in node["justifications"]: assert len(j["antecedents"]) == 1` loop in `test_any_creates_separate_justifications` is the strongest statement of this.
- **Outlist invariant**: expanded justifications are not stripped of `unless`; `outlist` is duplicated per branch.
- **premise_count semantics**:
  - `premise_count == 0` ⇔ node has no SL (premise node).
  - `premise_count == N` ⇔ one conjunctive SL with N antecedents (no `any_mode`).
  - `premise_count == 1` ⇔ `any_mode` was used (each JSL is unary).
  This mapping lets the CLI decide `if result["premise_count"] >= 3: warn(...)` regardless of which path was taken.
- **Hint eligibility**: `len(all_premises) > 1 AND len(surviving_premises) > 0 AND node != directly_retracted`. All three must hold; each is individually tested.
- **Composition with existing justifications**: `add_justification` *appends*; it never rewrites the original. `test_any_expands_add_justification` relies on `1 + 2 == 3`, not on replacing SL(a).
- **Derived truth**: after `retract_node`, `show_node(...)["truth_value"]` reflects the new state — the API is *not* lazy about recomputation.

---

## Error Handling

This file is a **happy-path suite** — no `pytest.raises`, no invalid input, no cycle/contradiction tests. Error semantics are therefore not asserted here; whatever `api.add_node` does with a malformed `sl` (e.g. missing node, empty string) is out of scope. That behavior lives in `tests/test_api.py`, `tests/test_add_justification.py`, etc.

Consequences for maintainers:
- If you introduce a stricter error path (e.g. `add_node` raises on unknown premise), these tests won't catch it — you must add coverage elsewhere.
- If a refactor makes `premise_count` silently missing from the return value, the tests fail — the contract is enforced.
- Some assertions are **brittle by design**: `hints[0]["surviving_premises"] == ["c"]` asserts exact list (order + content). If you ever return a `set`, this test fails. Same for `"a" in hint["all_premises"]` — `in` works with list, set, or dict; `== ["c"]` requires a sequence.
- Tests are inside a class but not `staticmethod`s; pytest instantiates each class per test method, so any future `__init__` state would need care (currently none).

---

## Topics to Explore

- [file] `reasons/api.py` — The module exercised here; read `add_node`, `add_justification`, and `retract_node` to see how `any_mode`, `premise_count`, and `restoration_hints` are actually computed.
- [function] `reasons/api.py:add_node` — Where the `any_mode` fan-out and `premise_count` derivation happen; check how `unless` is copied to each expanded justification.
- [function] `reasons/api.py:retract_node` — Where `restoration_hints` is computed; confirm the three gate conditions (multi-premise, surviving successor, not-self) are enforced in that order.
- [file] `reasons/storage.py` — The persistence layer beneath the API; useful to understand how justifications are keyed, since `add_justification(any_mode=True)` must append rather than overwrite.
- [general] `any-mode-vs-and-mode-semantics` — How `sl="a,b,c"` is interpreted as AND (one conjunctive JSL) vs. OR (three unary JSLs); this distinction drives the entire #20 feature and shapes the CLI UX.

## Beliefs

- `any-mode-expands-conjunction-into-separate-justifications` — `add_node(..., any_mode=True)` with an N-premise comma-separated `sl` produces N justifications, each containing exactly one antecedent.
- `any-mode-copies-outlist-to-each-expanded-justification` — When `unless="enemy"` is supplied alongside `any_mode=True`, every expanded justification contains `"enemy"` in its `outlist`.
- `add-node-returns-premise-count` — `api.add_node` returns a dict whose `"premise_count"` key holds the antecedent count for the created SL, and `0` for premise nodes.
- `any-mode-premise-count-is-one` — With `any_mode=True`, `add_node`'s returned `"premise_count"` is `1` (per unary justification), not the total number of expansions.
- `retract-node-returns-restoration-hints` — `api.retract_node` returns a dict with a `"restoration_hints"` list whose entries expose `node_id`, `all_premises`, and `surviving_premises`.
- `restoration-hints-skip-any-mode-nodes` — Nodes whose justifications are all unary (as produced by `any_mode`) never appear in `restoration_hints`, because the hint gate requires a multi-premise SL.
- `restoration-hints-exclude-directly-retracted-node` — The node passed to `retract_node` is never included in its own `restoration_hints`, even if it has multiple premises.
