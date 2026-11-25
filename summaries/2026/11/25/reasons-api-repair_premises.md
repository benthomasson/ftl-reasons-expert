---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: repair_premises in reasons/api.py

# `repair_premises` — Explanation

## 1. Purpose

`repair_premises` is the top-level, JSON-friendly API entry point for "fixing" premises whose content is factually wrong relative to their cited source. It is a two-stage pipeline:

1. **Gather review results** — either from a previously saved `review_premises` JSON report, or by re-reviewing a specified list of `belief_ids` inline.
2. **Repair only the flagged (inaccurate) premises** — the LLM-driven repair module produces a proposed action per premise (`rewrite` or `retract`); this function applies that plan (unless `dry_run=True`) and returns a summary.

The function exists so callers (CLI, LangGraph tools, scripts) don't need to manage `Storage` lifecycle, file resolution, or per-node mutation bookkeeping.

## 2. Contract

**Preconditions**
- Exactly one of `review_file` or `belief_ids` must be supplied (enforced with `ValueError`).
- If `review_file` is given, the file must exist and be valid JSON with a top-level `"results"` list whose entries contain an `"id"` field.
- If `belief_ids` is given, the DB at `db_path` must exist to allow `review_premises` (called with `dry_run=True`) to read the network.
- Each candidate premise must have a `source` field to be repairable; otherwise it is silently skipped.

**Postconditions**
- On non-`dry_run` success, for actions `"rewrite"` the old premise is superseded by a `-vN` node carrying the corrected text, and metadata `repair_action="rewritten"` is stamped. For `"retract"` the node is retracted and metadata `repair_action="retracted"` is stamped.
- The function always returns a dict, even when nothing is inaccurate.
- The return counters (`rewritten`, `retracted`, `failed`) sum <= `total_inaccurate`.

**Invariants implied but not enforced**
- `_repair` results are assumed to use `"action"` values of `"rewrite"` or `"retract"` (anything else creates no summary effect but is counted as neither rewrite nor retract).
- The review file entries are assumed to match the schema of `review_premises`' `"results"` list — specifically the `"accurate"` bool (treated as truthy default `True`).

## 3. Parameters

| Name | Type | Meaning | Edge cases |
|---|---|---|---|
| `review_file` | `str \| None` | Path to a saved `review_premises` JSON report | Mutually exclusive with `belief_ids`; missing file raises `FileNotFoundError`; malformed JSON raises `JSONDecodeError` |
| `belief_ids` | `list[str] \| None` | Re-review these premises inline then repair | Treated as truthy — empty list falls through to `review_file is None` → `ValueError` |
| `model` | `str` | LLM model name passed to both `review_premises` (re-review branch) and `_repair` | Must be a name understood by the project's `invoke_model`; wrong names surface from lower layers |
| `timeout` | `int` | Per-LLM-call timeout, forwarded to `_repair` and to `review_premises` when re-reviewing | Milliseconds vs seconds is determined downstream |
| `dry_run` | `bool` | If `True`, LLM repair still runs but no DB mutations occur | Does **not** skip the LLM call — just the applied changes |
| `parallel` | `int` | Parallelism for the `_repair` step | **Not forwarded** to `review_premises` in the inline branch (calls there are always serial) |
| `on_result` | `Callable \| None` | Callback invoked per repair result by `_repair` | Not forwarded to `review_premises` |
| `db_path` | `str` | Path to the SQLite `reasons.db` | Passed to `export_network`, `review_premises`, `supersede_with_text`, `retract_node`, `set_metadata` |

## 4. Return Value

Always a dict with these keys:

```python
{
  "results": [ ... ],          # raw results from _repair, mutated to add new_id/action=error
  "total_inaccurate": int,     # count of review results with accurate=False
  "rewritten": int,            # results whose final action == "rewrite"
  "retracted": int,            # results whose final action == "retract"
  "failed": int,               # results whose final action == "error"
}
```

Early-return path (nothing inaccurate): `{"results": [], "total_inaccurate": 0, "rewritten": 0, "retracted": 0, "failed": 0}`.

Callers must treat `rewritten + retracted + failed <= total_inaccurate` — items can be skipped earlier (no source, unreadable file) and never appear in `_repair`'s output, so the three counters may not sum to `total_inaccurate`.

**Important:** on a `dry_run`, the `failed` count and per-result mutation do not occur (the loop is skipped). However, `_repair` may still set `"action": "error"` in its own return value — those would appear in the counter sums for a dry run too because the counter logic applies regardless of `dry_run`.

## 5. Algorithm

1. Resolve input source:
   - If `review_file`: read JSON, `review_results_list = data.get("results", [])`.
   - Elif `belief_ids`: call `review_premises(belief_ids=..., model=..., timeout=..., dry_run=True, db_path=...)`, take `.get("results", [])`.
   - Else: `raise ValueError`.
2. Filter: `inaccurate = [r for r in review_results_list if not r.get("accurate", True)]`. If empty, return zero summary.
3. Build `review_map = {r["id"]: r for r in inaccurate}`. (Assumes every review entry has an `"id"`.)
4. `export_network(db_path=...)` to obtain `nodes` and `repos`. Convert `repos` values to `Path` objects; compute `db_dir = Path(db_path).parent`.
5. **Resolve source contents** (cached per source string):
   - Skip nodes with no `source` or `source not in nodes` (silent).
   - Get `agent = metadata.get("agent")` to support agent-namespaced source roots.
   - `resolve_source_path(source, repos=repos, db_dir=db_dir, agent=agent)`; if the resulting path doesn't exist → print SKIP to stderr and skip.
   - Read file; on `OSError`/`UnicodeDecodeError` → skip silently.
   - Use cache dict `source_contents[source]` so identical sources produce a single disk read.
   - Accumulate `repair_ids` for premises whose source was successfully read.
6. Call `_repair(nodes, premise_ids=repair_ids, source_contents=..., review_results=review_map, model=..., timeout=..., parallel=..., on_result=...)`.
7. If `not dry_run`, for each result:
   - `action == "rewrite"` and `corrected_text` present: `supersede_with_text(nid, corrected_text, db_path=...)`, store `r["new_id"]`, then `set_metadata(new_id, "repair_action", "rewritten")`.
   - `action == "retract"`: `retract_node(nid, reason=f"repair-premises: {rationale}", db_path=...)`, then `set_metadata(nid, "repair_action", "retracted")`.
   - Any exception in either branch: print `ERROR ...` to stderr and set `r["action"] = "error"`.
8. Count results using the final `action` value; return summary.

## 6. Side Effects

- **LLM calls**: both `review_premises` (inline-review branch) and `_repair` internally invoke the project's LLM, incurring API cost and latency. Both are `dry_run` in terms of DB writes for `review_premises` (forced `dry_run=True`), but `_repair` still calls the model.
- **Filesystem reads**: JSON `review_file`; every unique source file discovered in step 5 (each read once per unique `source` string).
- **Database mutations** (non-`dry_run` only):
  - `supersede_with_text` — creates a new node (`{old}-vN` by default), links via supersession outlist, recomputes truth values, saves DB.
  - `retract_node` — retracts the premise and cascades, saving DB.
  - `set_metadata` — writes `repair_action` into the replaced/original node.
- **stderr I/O**: per-skip `SKIP ...` and per-error `ERROR ...` messages.

## 7. Error Handling

- **Raised (uncaught) to the caller:**
  - `ValueError` when neither `review_file` nor `belief_ids` is provided.
  - `FileNotFoundError` from reading `review_file`.
  - `json.JSONDecodeError` from a malformed `review_file`.
  - Exceptions inside `_repair` (LLM failures, malformed `_repair` output) — there is no try/except around the `_repair` call.
- **Swallowed (per-item):**
  - Missing/unreadable sources are skipped silently (missing source) or logged (`source not found`).
  - Exceptions from `supersede_with_text`, `retract_node`, or `set_metadata` are caught and the result action becomes `"error"`, but any earlier side effect (e.g., a successful supersede) is **not rolled back**. This is a latent inconsistency: a `rewrite` may have been applied but the summary counts it as failed.
- **Implicit contract errors:** entries in the review output without an `"id"` key would raise `KeyError` at `{r["id"]: ...}`. Entries without `"accurate"` default to accurate (not repaired).

## 8. Usage Patterns

Typical invocation (CLI-like):

```python
# From a saved review
res = repair_premises(review_file="review-premises.json", dry_run=False)

# Re-review and repair a specific set
res = repair_premises(belief_ids=["belief-42", "belief-7"], dry_run=True)

# Triage without changes
res = repair_premises(review_file="review.json", dry_run=True)
```

Caller obligations:
- Always pair with a prior `review_premises` run (or trust that the file schema matches), because there is no schema validation on the review file — the function assumes `"results"` is a list of dicts with `"id"` and `"accurate"`.
- For rewrites, decide how to handle the fact that the original premise is **superseded, not edited in place**. Downstream dependents survive via the supersession outlist mechanism (`old goes OUT when new goes IN`) but the node ID changes (`-vN`).
- Inspect `res["results"]` to distinguish genuine successes from the "applied but reported as error" edge case described above.

## 9. Dependencies

- `reasons.review_premises.review_premises` — the LLM review that produces the input artifact.
- `reasons.repair_premises.repair_premises` — the LLM repair driver (`_repair`) that decides `rewrite` vs `retract`.
- `reasons.check_stale.resolve_source_path` — maps a `source` string to a filesystem path using `repos` and `db_dir`, honoring an `agent` prefix.
- `reasons.api` sibling functions: `export_network`, `supersede_with_text`, `retract_node`, `set_metadata`, `review_premises`.
- Stdlib: `json`, `pathlib.Path`, `sys`.
- `reasons.storage.Storage` indirectly, via `_with_network` used by all mutating helpers.

## Topics to Explore

- [file] `reasons/repair_premises.py` — The actual repair logic (LLM prompt, `action` decisions, `corrected_text` generation) that this wrapper delegates to
- [function] `reasons/api.py:review_premises` — How the review artifact this function consumes is produced (including the `accurate`/`well_scoped`/`error_type` fields)
- [function] `reasons/api.py:supersede_with_text` — The reuse/preservation mechanism that lets derived beliefs survive a premise rewrite
- [function] `reasons/check_stale.py:resolve_source_path` — Resolution rules for mapping a `source` string to a real path (repos + agent prefix)
- [file] `tests/test_repair_premises.py` — Authoritative examples of the review file schema and expected repair outcomes

## Beliefs

- `repair-premises-requires-input` — `repair_premises` raises `ValueError` when neither `review_file` nor a truthy `belief_ids` is provided.
- `repair-premises-dry-run-prevents-mutation` — When `dry_run=True`, no `supersede_with_text`, `retract_node`, or `set_metadata` call is made against the database.
- `repair-premises-parallel-not-forwarded` — The `parallel` parameter is forwarded to `_repair` but not to the inline `review_premises` call.
- `repair-premises-reports-error-after-applied-change` — A successful `supersede_with_text`/`retract_node` followed by a failing `set_metadata` results in `"action": "error"` even though the DB was mutated.
- `repair-premises-filters-on-accurate` — Only review results whose `accurate` field is falsy are considered for repair; missing `accurate` defaults to `True` and is skipped.
