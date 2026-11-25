---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: repair_smuggled in reasons/api.py

# `repair_smuggled` — Explanation

```python
def repair_smuggled(
    review_file: str | None = None,
    belief_ids: list[str] | None = None,
    model: str = "claude",
    timeout: int = 300,
    dry_run: bool = False,
    db_path: str = DEFAULT_DB,
) -> dict:
```

## Purpose

`repair_smuggled` is a thin orchestration layer over the actual repair logic in `reasons/repair.py:repair_smuggled_beliefs`. It is the public entrypoint a CLI command, LangGraph tool, or script can call to fix beliefs that were flagged **invalid** by `review_beliefs` due to *smuggled premises* — conclusions that rely on assumptions not explicitly present as antecedents in the belief's justification graph.

The function exists to (a) normalize the two ways a caller might supply "which beliefs need repairing" (a saved review report vs. an inline re-review), and (b) produce a stable, JSON-serializable summary of the repair pass.

## Contract

**Preconditions**
- Exactly one of `review_file` / `belief_ids` should be supplied. If `review_file` is truthy, it wins; `belief_ids` is ignored (uses `elif`, not `or`).
- If `review_file` is used, it must be a path to a JSON file shaped like the output of `review_beliefs` — specifically, a top-level object with a `"results"` key holding a list of review records.
- If `belief_ids` is used, each ID must resolve to a node in the network (otherwise `review_beliefs` will simply get an empty filtered set).
- `db_path` must point to an existing RMS SQLite database, since `export_network` is called on it.

**Postconditions**
- Always returns a dict with exactly six keys: `repairs`, `total_invalid`, `repaired`, `no_candidates`, `no_match`, `extraction_failed`, `errors`.
- If no invalid reviews are found, the function short-circuits with an empty `repairs` list and all zero counts — never invokes the repair layer.
- With `dry_run=True`, `review_beliefs` is invoked with `dry_run=True` and the delegate also receives `dry_run=True`; no DB mutations should result (the delegate is trusted to honor this).
- Individual *repair* failures do not raise — they are captured as a `status: "error"` entry inside the delegate's result list and counted.

**Invariants**
- The function never opens `_with_network(write=True)` itself. All database writes, if any, happen inside `repair_smuggled_beliefs`.
- The set of status strings it recognizes (`repaired`, `no_candidates`, `no_match`, `extraction_failed`, `error`) is hardcoded and must match what the delegate emits.

## Parameters

| Parameter | Type | Meaning / Edge cases |
|---|---|---|
| `review_file` | `str \| None` | Path to a previously-saved review report JSON. Truthy value short-circuits `belief_ids`. File must exist and parse as JSON with a `results` list. Missing `results` key silently yields `[]` (see `.get("results", [])`). |
| `belief_ids` | `list[str] \| None` | Belief node IDs to re-review and repair inline. An **empty list is falsy** — `repair_smuggled(belief_ids=[])` raises `ValueError` (falls through to the `else`). Combined with `review_file`, the file wins. |
| `model` | `str` | LLM identifier passed verbatim to `review_beliefs` and then to the repair delegate. Default `"claude"`. |
| `timeout` | `int` | LLM call timeout in seconds. |
| `dry_run` | `bool` | If `True`, no repairs are applied. If `False`, the delegate is expected to persist changes. |
| `db_path` | `str` | SQLite path; default `DEFAULT_DB` (`"reasons.db"`). Must already exist. |

## Return Value

```python
{
    "repairs": list[dict],      # per-repair results from repair_smuggled_beliefs
    "total_invalid": int,       # len of invalid reviews fed to the delegate
    "repaired": int,            # count where r["status"] == "repaired"
    "no_candidates": int,       # count where r["status"] == "no_candidates"
    "no_match": int,            # count where r["status"] == "no_match"
    "extraction_failed": int,   # count where r["status"] == "extraction_failed"
    "errors": int,              # count where r["status"] == "error"
}
```

The caller must be prepared for a **zero-result short circuit** (all zeros, empty `repairs`) when no belief was flagged invalid. Any per-item failure in the delegate surfaces as `errors > 0` plus individual `"error"` entries — it is *not* raised.

## Algorithm

1. **Import lazily.** `from .repair import repair_smuggled_beliefs` — avoids a circular import at module load time.
2. **Load reviews.**
   - If `review_file`: `json.load` it and read `report.get("results", [])`.
   - Elif `belief_ids`: call `review_beliefs(belief_ids=..., model=..., timeout=..., dry_run=True, db_path=...)` and take `result["results"]`. Note `dry_run=True` — the review step doesn't mutate the DB.
   - Else: raise `ValueError("Provide either review_file or belief_ids")`.
3. **Filter to invalid.** `invalid = [r for r in review_results if not r.get("valid", True)]`. Records with `valid=True` **or a missing `valid` key** are treated as valid and skipped.
4. **Short-circuit** if `invalid` is empty (avoids an expensive `export_network` + LLM call).
5. **Export the network.** `net = export_network(db_path=db_path)` returns the full dict representation (nodes with text, justifications, metadata, etc.). `nodes = net.get("nodes", {})`.
6. **Delegate.** Call `repair_smuggled_beliefs(invalid, nodes, model=..., timeout=..., db_path=..., dry_run=...)`. This is where the actual "search for existing premises to link as additional antecedents" work happens.
7. **Aggregate.** Build the summary dict by counting `status` fields on each delegate result. Unknown statuses are silently ignored by the `sum(...)`.

## Side Effects

- **Disk reads:** `review_file` is opened and parsed when used.
- **DB reads:** `review_beliefs` (when re-reviewing) and `export_network` both read the SQLite database. Neither opens a write context in this function.
- **DB writes:** delegated. If `dry_run=False` *and* `repair_smuggled_beliefs` decides to apply a repair (typically `add_justification`, `supersede_with_text`, or a metadata stamp), those write through the delegate's own `_with_network(write=True)` blocks. This function does **not** wrap the whole run in a single transaction — a mid-batch exception will propagate and leave whatever the delegate already wrote in place.
- **LLM calls:** one via `review_beliefs` (when `belief_ids`) plus any calls the repair delegate makes.

## Error Handling

**Raises**
- `ValueError` — neither `review_file` nor `belief_ids` supplied, or `belief_ids=[]`.
- `FileNotFoundError` / `OSError` — `review_file` path unreadable.
- `json.JSONDecodeError` — `review_file` is not valid JSON.
- Whatever `review_beliefs` / `export_network` / `repair_smuggled_beliefs` raise (e.g. `KeyError` on missing node, LLM transport errors, `sqlite3.OperationalError` on a corrupt/missing DB).

**Swallowed / surfaced as data**
- Per-repair failures are **not** raised; they become `{"status": "error", ...}` items and are counted in `errors`.
- A `review_file` without a top-level `results` key silently yields zero invalid reviews — this is a silent-success trap, not an error.

## Usage Patterns

```python
# 1. Post-hoc: repair everything review_beliefs flagged as invalid
report = review_beliefs(model="claude", db_path="reasons.db")
Path("review.json").write_text(json.dumps(report))
summary = repair_smuggled(review_file="review.json", dry_run=True)
if summary["repaired"] and not summary["errors"]:
    repair_smuggled(review_file="review.json", dry_run=False)

# 2. Ad-hoc: re-review a small set inline and repair
repair_smuggled(belief_ids=["claim-42", "claim-99"], dry_run=False)
```

**Caller obligations**
- Prefer a `dry_run=True` pass first to inspect `repairs` and `errors` before committing.
- Do not assume the call is atomic — if `dry_run=False` and something raises mid-batch, partial repairs may already be applied.
- Never rely on `review_file` validation; produce it from `review_beliefs` directly.

## Dependencies

- `.repair.repair_smuggled_beliefs` — the actual repair engine (`reasons/repair.py`).
- `reasons.api.review_beliefs` — same module; produces the review report shape this function consumes.
- `reasons.api.export_network` — same module; snapshot of the network passed to the repair engine.
- `json` (stdlib) — review-file parsing.
- Indirect: `reasons.review`, `reasons.llm`, `reasons.storage`, and the LLM provider itself.

## Assumptions Not Enforced by Types

- `review_file` JSON must match `review_beliefs`' output schema (dict with `results` list of `{id, valid, ...}`). Any other shape silently resolves to an empty review set and a "no-op success."
- Delegate repair results **must** contain a `status` key that is one of the five expected strings. Typos or new statuses silently vanish from the summary counts while still appearing in `repairs`.
- `db_path` is assumed to already exist — the function does not call `init_db`.
- If both `review_file` and `belief_ids` are passed, `belief_ids` is discarded without warning.

---

## Topics to Explore

- [function] `reasons/repair.py:repair_smuggled_beliefs` — The actual repair algorithm: how it searches for candidate premises and how it decides between `repaired`, `no_match`, and `no_candidates`.
- [function] `reasons/api.py:review_beliefs` — Defines the exact review-report schema (`results`, `invalid`, `sufficient`, `necessary`) that `repair_smuggled` consumes via `review_file`.
- [function] `reasons/api.py:export_network` — The node dict shape (`justifications`, `truth_value`, `metadata`, `dependents`) the delegate must parse.
- [file] `tests/test_repair.py` — Exercises the smuggled-premise repair paths; likely shows fixture formats for `review_file` and the exact status strings expected.
- [general] `smuggled-premise-detection` — Background on what a "smuggled premise" is and how the review LLM decides `valid=False`, to understand why repair tries to link existing premises rather than rewrite text.

## Beliefs

- `repair-smuggled-delegates-to-repair-module` — All actual repair work happens in `reasons/repair.py:repair_smuggled_beliefs`; `repair_smuggled` is only an adapter for input normalization and summary aggregation.
- `repair-smuggled-requires-review-input` — The function raises `ValueError` if neither `review_file` nor a non-empty `belief_ids` is supplied.
- `repair-smuggled-filters-by-valid-flag` — Only review entries where `valid` is falsy are repaired; missing `valid` keys are treated as valid and skipped.
- `repair-smuggled-trusts-review-json-shape` — A `review_file` lacking a top-level `results` key returns a zero-count summary instead of raising, masking the erroneous input.
- `repair-smuggled-dry-run-is-not-transactional` — `dry_run=False` does not wrap the whole run in a single transaction; partial writes may persist if the delegate raises mid-batch.
