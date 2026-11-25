---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: repair in reasons/api.py

# `reasons.api.repair`

## Purpose

`repair` is a top-level entry point on the RMS Python API for **repairing beliefs that an earlier review pass flagged as invalid**. It is invoked by the CLI (`reasons repair …`) as well as by other scripts and LLM tools that want to programmatically act on a review report.

The function doesn't itself contain repair logic — it is a thin orchestration wrapper that:

1. obtains review results (either from a saved JSON report or by re-running `review_beliefs` inline),
2. filters down to the *invalid* beliefs,
3. fetches a network snapshot via `export_network`, and
4. delegates the actual triage/rewrite/retract work to `.repair.repair_beliefs`, aggregating its per-belief statuses into a summary dict.

At module level the function is also assigned to the alias `research = repair`, so `reasons.api.research` and `reasons.api.repair` are literally the same function object. That alias exists so the CLI can present the same call under two different intents without duplicating logic.

## Contract

**Preconditions**

- Either `review_file` **or** `belief_ids` must be *truthy* (see caveat below). Otherwise a `ValueError` is raised.
- `review_file`, if given, must exist and contain JSON with a top-level `"results"` list. Each entry is expected to be a per-belief review dict containing at least a `"valid"` boolean (anything not literally `False` is treated as valid via `r.get("valid", True)`).
- `belief_ids`, if used, are re-reviewed with `review_beliefs(..., dry_run=True)` — so any node IDs that don't correspond to derived/IN beliefs are silently dropped by that lower-level function, not here.
- `db_path` must point at a valid RMS database for `export_network` and `repair_beliefs`.

**Postconditions**

- Returns a dict with `results` (list of per-belief repair records) plus aggregate counts: `total_invalid`, `linked`, `softened`, `abandoned`, `needs_research`, `failed`, `errors`.
- When the review contains zero invalid beliefs, the function returns an all-zero dict **without** calling `export_network` or `.repair.repair_beliefs` — a fast short-circuit.
- The `total_invalid` count is `len(invalid)` *from the review results*, not `len(results)`. If `repair_beliefs` ever drops or skips inputs (unlikely but not type-enforced), the counts and `total_invalid` could disagree.

**Invariants / assumptions the type system does not enforce**

- `review_file` takes precedence: if both are supplied, `belief_ids` is silently ignored.
- An **empty list** for `belief_ids` is falsy — it falls through to `raise ValueError`, since the dispatch is `elif belief_ids:`. Callers that want a no-op must pass `review_file` or a non-empty list.
- Every status returned by `.repair.repair_beliefs` is *expected* to be one of `{linked, softened, abandoned, needs_research, triage_failed, no_candidates, no_match, soften_failed, extraction_failed, error}`. If the module returns any other status string, the aggregate counters will undercount.
- `dry_run` affects only the repair step, not the review step: inline review is always run with `dry_run=True`.

## Parameters

| name | type | meaning | edge cases |
|---|---|---|---|
| `review_file` | `str \| None` | Path to a saved `review_beliefs` JSON report. | File-not-found → `FileNotFoundError`; malformed JSON → `json.JSONDecodeError`; missing `"results"` key → `review_results == []` (treated as *no invalid*, early return). |
| `belief_ids` | `list[str] \| None` | Beliefs to re-review inline. Ignored when `review_file` is set. | Empty list is falsy → falls through to `ValueError`. Individual IDs that aren't derived/IN beliefs are pruned internally by `review_beliefs`. |
| `model` | `str = "claude"` | LLM model name passed to both `review_beliefs` (inline mode) and `repair_beliefs`. | Not validated locally. |
| `timeout` | `int = 300` | Per-call LLM timeout (seconds). | Passed straight through. |
| `dry_run` | `bool = False` | If `True`, `.repair.repair_beliefs` should compute repairs without mutating the DB. | The *review* re-run is always dry regardless. |
| `db_path` | `str = DEFAULT_DB` (`"reasons.db"`) | Path to the SQLite RMS database. | Used both for `export_network` and passed to `repair_beliefs` so it can open its own write context. |

## Return Value

Two shapes, depending on whether any invalid beliefs were found.

**Empty-repair shape** (when `invalid == []`):
```python
{
    "results": [],
    "total_invalid": 0,
    "linked": 0,
    "softened": 0,
    "abandoned": 0,
    "needs_research": 0,
    "failed": 0,
    "errors": 0,
}
```

**Normal shape**:
```python
{
    "results": [<per-belief repair dict>, ...],
    "total_invalid": N,
    "linked": <count of status=="linked">,
    "softened": <count of status=="softened">,
    "abandoned": <count of status=="abandoned">,
    "needs_research": <count of status=="needs_research">,
    "failed": <count of triage_failed/no_candidates/no_match/soften_failed/extraction_failed>,
    "errors": <count of status=="error">,
}
```

Caller obligations:
- Do not assume `total_invalid == linked + softened + abandoned + needs_research + failed + errors` unless `.repair.repair_beliefs` is verified to be exhaustive.
- `results` items are dicts owned by the `.repair` module; the shape of each element is not described by this function's type hints (still `dict`, but fields vary by status).
- When `dry_run=True`, the caller is expected to inspect `results` themselves and follow up with a real (non-dry) call to actually apply.

## Algorithm (walk-through)

1. **Import** `repair_beliefs` from `.repair`.
2. **Resolve review results**:
   - If `review_file` is truthy: `open(...)` and `json.load(...)`, then take `.get("results", [])`.
   - Elif `belief_ids` is truthy: call `review_beliefs(belief_ids=..., model=..., timeout=..., dry_run=True, db_path=...)` and take `result["results"]`. (Note: this call is intentional `dry_run=True` — we only want review classifications here; actual mutation is the `repair_beliefs` step's job.)
   - Else: `raise ValueError("Provide either review_file or belief_ids")`.
3. **Filter to invalid**: `invalid = [r for r in review_results if not r.get("valid", True)]`. The `.get(..., True)` default means a *missing* `valid` key counts as valid — so only entries with an explicit `"valid": False` (or any falsy `valid`) end up in `invalid`.
4. **Short-circuit**: if `invalid` is empty, return the all-zero summary. No network is loaded, no LLM is called, no mutation occurs.
5. **Snapshot the network**: `export_network(db_path=db_path)` — this returns a dict with a `"nodes"` mapping. Only `nodes` is kept.
6. **Delegate**: `repair_beliefs(invalid, nodes, model=..., timeout=..., db_path=..., dry_run=dry_run)`. This is where the four-way triage (link / soften / abandon / needs_research) lives. Because `repair` passes `db_path` and `dry_run` through, `repair_beliefs` is responsible for *applying* changes (supersessions, retractions, softening text) when `dry_run=False` — it opens its own write context internally.
7. **Aggregate**: walk `results` and sum each status counter with `sum(1 for r in results if r["status"] == ...)`. Statuses classified as "failed" are a hard-coded tuple: `triage_failed`, `no_candidates`, `no_match`, `soften_failed`, `extraction_failed`. Everything else fires the "error" counter if the status is exactly `"error"`.
8. **Return** the summary dict.

## Side Effects

- **File I/O**: reads `review_file` when provided (no writes).
- **Database read**: `export_network` opens `db_path` in read mode and returns a serialized snapshot.
- **LLM calls**: via `review_beliefs` when `belief_ids` is the input mode; via `repair_beliefs` unconditionally (repair involves LLM reasoning).
- **Database writes**: only if `dry_run=False` **and** at least one repair produces an actionable result. What the actual writes are depends entirely on `.repair.repair_beliefs` (typically: `add_node` for replacements, `supersede` / `supersede_with_text`, `retract_node`, `set_metadata`). `repair` itself never calls those directly.
- **No global state**: doesn't touch `_network_cache`, doesn't clear caches; each downstream call manages its own.

## Error Handling

Raises (not caught):
- `ValueError` — neither input mode supplied (including the empty-`belief_ids` case).
- `FileNotFoundError` — `review_file` path missing.
- `json.JSONDecodeError` — `review_file` isn't valid JSON.
- Any exception from `export_network` or `repair_beliefs` propagates unchanged.

Returned rather than raised:
- Per-belief failures *inside* repair are captured as `status == "error"` (and counted under `errors`) or as one of the `failed` statuses. That means the caller normally sees a successful function call even if some beliefs failed to repair — failures are reflected in the counters.

Swallowed:
- Nothing in this function body invokes `try/except`. All error handling of individual repairs is delegated to `.repair.repair_beliefs`.

## Usage Patterns

Typical CLI-style usage:
```python
# From a saved review report — no re-review
repair(review_file="review-out.json", dry_run=False)

# Re-review specific beliefs and repair inline
repair(belief_ids=["b1", "b2", "b3"], model="claude")

# Preview only — no DB writes
repair(review_file="review-out.json", dry_run=True)
```

Caller obligations:
- Provide exactly one of the two inputs; do not pass an empty `belief_ids` list.
- Inspect `results` even when `dry_run=False`, because some beliefs may end up as `failed` / `errors` even though the overall call succeeded.
- The `research = repair` alias means any test that monkeypatches `reasons.api.repair` must also patch `reasons.api.research` (and vice-versa) unless a single point of indirection is used.

## Dependencies

- `.repair.repair_beliefs` — the core repair engine (imported lazily inside the body).
- `.api.review_beliefs` (same module) — the inline re-review path.
- `.api.export_network` (same module) — provides the `nodes` snapshot passed to `repair_beliefs`.
- `json` (stdlib) — for parsing `review_file`.
- `.review` module (indirectly, through `review_beliefs`) — the underlying LLM review logic.

## Topics to Explore

- [file] `src/reasons/repair.py` — The actual repair engine; defines the status vocabulary (`linked`/`softened`/`abandoned`/`needs_research`) that this wrapper aggregates. Understanding its `dry_run` semantics is critical.
- [function] `src/reasons/api.py:review_beliefs` — Determines which beliefs get flagged as `valid: False` in the first place; its `include_out`/`min_depth`/`depends_on` filters interact with `belief_ids`.
- [function] `src/reasons/api.py:repair_smuggled` — Sibling function using an almost identical orchestration pattern; comparing the two shows the intended template for "review then act" APIs.
- [file] `src/reasons/api.py` — The alias `research = repair` is defined at module scope; check CLI wiring and `tests/test_research.py` to see how the two intents differ at the surface.
- [general] `dry-run-propagation` — Trace where `dry_run` is honored vs. ignored (inline review is always dry; repair must honor the flag) and verify nothing mutates the DB in a `dry_run=True` call.

## Beliefs

- `repair-requires-exactly-one-input-mode` — `repair` raises `ValueError` unless exactly one of `review_file` or a truthy `belief_ids` is provided; both may be set but `review_file` wins.
- `empty-belief-ids-raises-value-error` — Passing `belief_ids=[]` behaves like passing nothing because the dispatch uses `elif belief_ids:`, so an empty list triggers the `ValueError`.
- `repair-delegates-mutation-to-repair-belifes` — `repair` never mutates the DB directly; all writes (supersede/retract/set_metadata) happen inside `.repair.repair_beliefs`, driven by the `db_path`/`dry_run` arguments it forwards.
- `inline-review-is-always-dry` — When `belief_ids` is the input mode, `review_beliefs` is called with `dry_run=True` unconditionally; the caller's `dry_run` only affects the subsequent repair stage.
- `research-is-repair-alias` — At module scope, `research = repair`, so both names resolve to the same function object and monkeypatching one affects the other.
