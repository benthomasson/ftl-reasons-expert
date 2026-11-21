---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: research in reasons/api.py

# Explanation: `research = repair` in `reasons/api.py`

## Purpose

This single line is a **module-level name alias**. It binds the name `research` to the same function object that `repair` refers to at the moment this statement executes. No wrapper, no helper, no new function is created.

The alias exists so that callers/CLI commands/agents that think in terms of a "research" workflow can invoke `research(...)` while the underlying engine is the same code path as `repair(...)`. The chosen name is motivated by the fact that `repair` produces a `"needs_research"` status for flagged beliefs (see `repair`'s return handoff to `repair_beliefs` from `reasons/repair.py`), so `research` is a semantically equivalent entry point for callers whose mental model is "go research this belief."

## Contract

- **Preconditions**: None at import time. The assignment requires that `repair` already exists in the module namespace (it does, defined immediately above).
- **Postconditions**: `reasons.api.research is reasons.api.repair` evaluates to `True` at import time. `research.__name__`, `research.__doc__`, `research.__code__`, and `research.__defaults__` all belong to the `repair` function object unchanged.
- **Invariants** (assumed, not enforced):
  - `repair` must not be rebound after this line (it isn't). If a later line reassigned `repair` to a different object, `research` would still point at the *old* function — the alias captures the binding, not a live reference to the name.
  - The alias's behavior and signature are *identical* to `repair`; it inherits every parameter, exception, and side effect.

## Parameters

`research` accepts exactly the parameters of `repair`:

| Param | Type | Meaning |
|---|---|---|
| `review_file` | `str \| None` | Path to a JSON report produced by `review_beliefs`; `report["results"]` is extracted. |
| `belief_ids` | `list[str] \| None` | Node IDs to re-review inline (dry-run) if `review_file` is not provided. |
| `model` | `str` | LLM model identifier (default `"claude"`). |
| `timeout` | `int` | Per-call LLM timeout in seconds (default `300`). |
| `dry_run` | `bool` | If `True`, repairs are classified but not persisted. |
| `db_path` | `str` | Path to the SQLite RMS database (default `DEFAULT_DB`). |

**Edge cases**:
- Passing *both* `review_file` and `belief_ids` → `review_file` wins (the `elif` branch is never reached).
- Passing *neither* → raises `ValueError("Provide either review_file or belief_ids")`.
- Passing `review_file` for a nonexistent path → `FileNotFoundError` from `open(...)`.
- Passing `review_file` for malformed JSON → `json.JSONDecodeError`.

Not enforced by types:
- The review report format (`results` list with `id` and `valid` keys) is assumed but not validated.
- The LLM `model`/`timeout` are forwarded to the `.repair` module and then to the LLM layer; behavior there is unspecified from this file's perspective.

## Return Value

`research(...)` returns the exact dict that `repair(...)` returns:

**Empty case** (no invalid results): a dict with all counter keys set to zero:
```python
{
    "results": [], "total_invalid": 0, "linked": 0,
    "softened": 0, "abandoned": 0, "needs_research": 0,
    "failed": 0, "errors": 0,
}
```

**Populated case**:
```python
{
    "results": [...],          # per-belief outcomes from repair_beliefs
    "total_invalid": int,      # count of invalid reviews considered
    "linked": int,             # status == "linked"
    "softened": int,           # status == "softened"
    "abandoned": int,          # status == "abandoned"
    "needs_research": int,     # status == "needs_research"
    "failed": int,             # sum of triage_failed/no_candidates/no_match/soften_failed/extraction_failed
    "errors": int,             # status == "error"
}
```

Caller obligations:
- Must handle `ValueError` when neither input is supplied.
- Must be aware that `dry_run=True` returns one result dict per invalid belief but performs no database mutations — callers wishing to persist must re-run with `dry_run=False`.
- Must handle `needs_research` entries explicitly — they represent beliefs the LLM could not resolve locally and which likely require external retrieval.

## Algorithm

Because `research` **is** `repair`, the algorithm is precisely:

1. **Input selection**: prefer `review_file` (parse JSON, take `report["results"]`), otherwise re-run `review_beliefs(..., dry_run=True)` on `belief_ids` to generate `review_results`.
2. **Filter**: `invalid = [r for r in review_results if not r.get("valid", True)]`.
3. **Early exit**: if `invalid` is empty, return the zeroed summary dict (no database I/O).
4. **Snapshot**: `net = export_network(db_path=db_path)`, `nodes = net.get("nodes", {})` — a read-only snapshot of the network so the repair module can inspect justified beliefs without mutating state.
5. **Delegate**: call `repair_beliefs(invalid, nodes, model, timeout, db_path, dry_run)` from `reasons/repair.py`, which triages each invalid belief into `linked`, `softened`, `abandoned`, or `needs_research` (with failure statuses for exceptions).
6. **Aggregate**: compute counts per status and return the summary dict.

## Side Effects

**At import time (the alias line itself)**:
- None beyond binding the name `research` in `reasons.api`'s module namespace. No I/O, no allocation, no external calls.

**At call time**:
- Inherits all side effects of `repair`:
  - Filesystem read of `review_file` (if used).
  - LLM network calls via `review_beliefs` and `repair_beliefs` (unless `dry_run=True` and no repair status writes up the stack).
  - Possible database writes via `reasons/repair.py` when `dry_run=False` (supersede/retract calls through `supersede_with_text`, `retract_node`, etc., which use `_with_network(db_path, write=True)`).

## Error Handling

`research` inherits `repair`'s exception surface exactly:

- `ValueError` — neither `review_file` nor `belief_ids` provided.
- `FileNotFoundError` — `review_file` path does not exist.
- `json.JSONDecodeError` — malformed review report.
- Anything raised by `export_network` (e.g., `sqlite3.DatabaseError` on a corrupt DB).
- Anything raised inside `repair_beliefs` (LLM failures, DB write failures).
- `dry_run=True` swallows write failures in `repair_beliefs` by design (no writes attempted).

`repair` does **not** wrap this in a try/except, so failures from the delegate propagate. Callers that need graceful degradation must implement it themselves.

Non-obvious: the `errors` count in the return value is *only* for status == `"error"` from `repair_beliefs`, not for exceptions — exceptions are not counted, they propagate.

## Usage Patterns

Typical patterns (identical for `research` and `repair`):

```python
# Re-review a set of beliefs and repair the invalid ones:
from reasons.api import research
result = research(belief_ids=["belief-1", "belief-2"], model="claude")
print(result["linked"], result["needs_research"])

# Feed a pre-computed review report:
result = research(review_file="reports/review-2024.json", dry_run=True)

# Research vs repair is a naming choice only:
from reasons.api import repair, research
assert repair is research   # True at import time
```

**Caller obligations**:
- Choose one of `review_file` or `belief_ids`; do not pass neither.
- Inspect `needs_research` in `results` explicitly — this is the "external retrieval required" bucket, distinct from `errors`/`failed`.
- Do not rely on `research` being a *wrapper* — it has no added validation, logging, prompt, or namespace. If you need distinct semantics (e.g., "research does external web lookup"), you must implement it separately; the current alias provides none of that.

## Dependencies

- **Internal**: `reasons.api.repair_beliefs` (the LLM-based reviewer), `reasons.api.export_network`, `reasons.api.what_if_*` (not called here but used by sibling functions), `reasons/repair.py:repair_beliefs` (the delegated implementation).
- **Stdlib**: `json` (for review_file parsing), `pathlib` (indirect via `open`), plus the myriad imports at the top of `api.py`.
- **External**: LLM provider via `reasons/llm.py`, indirectly through `repair_beliefs`.
- **No new dependency** is introduced by the alias line itself.

## Non-Obvious Notes and Assumptions

- The alias is **not** a `def research(...): return repair(...)` wrapper. Tests are free to assert `reasons.api.research is reasons.api.repair`; this is a stronger contract than a wrapper would give.
- `research.__name__` still reports `"repair"`, and `inspect.signature(research)` is `repair`'s signature — CLI frameworks that auto-generate flags from `__name__`/docstring will render "repair," not "research."
- If someone later rebinds `repair` (e.g., monkeypatches it, or a refactor introduces `def repair_v2` and assigns `repair = repair_v2`), `research` will still point to the **original** `repair` object. This is a silent drift hazard: aliasing captures the object, not the name.
- There is no `__all__` in `api.py`, so both names are publicly reachable via `from reasons.api import research`.
- The alias is colocated with `repair` (both near line `research = repair`), which is good practice — keep them adjacent to avoid stale aliases.

## Topics to Explore

- [file] `reasons/repair.py` — Implements `repair_beliefs`, the triage engine that classifies invalid beliefs as `linked`, `softened`, `abandoned`, or `needs_research`.
- [function] `reasons/api.py:repair` — The aliased function; its behavior is the actual contract of `research`.
- [function] `reasons/api.py:repair_smuggled` — Sibling function that handles only the "smuggled premise" case; contrast with `repair`'s broader triage.
- [file] `tests/test_research.py` — Confirms expectations for the `research` alias (likely asserts identity or behavioral equivalence with `repair`).
- [general] `research-alias-design` — Why a bare alias was chosen instead of a wrapper (`__name__`/`__doc__`/signature preservation, import-time cost, test surface).

## Beliefs

- `research-is-repair` — At import time, `reasons.api.research is reasons.api.repair` evaluates to `True`; the alias binds the same function object.
- `research-preserves-repair-name` — `research.__name__ == "repair"` and `research.__doc__` are the repair docstring; no renaming occurs.
- `research-has-no-wrapper` — No additional validation, logging, or logic exists between the two names; `research(...)` is bit-for-bit identical to `repair(...)`.
- `repair-produces-needs-research-status` — The `repair` return dict includes a `needs_research` counter, motivating the `research` alias for callers whose workflow is "resolve external evidence."
- `research-raises-on-missing-input` — Calling `research()` with neither `review_file` nor `belief_ids` raises `ValueError`, not a return-dict error status.
- `alias-captures-object-not-name` — If `repair` were rebound later in the module, `research` would still point at the original function object captured at alias time.
