---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: repair_smuggled_beliefs in reasons/repair.py

## Purpose
`repair_smuggled_beliefs` is a specialized repair orchestrator for invalid beliefs whose problem is a missing antecedent / smuggled premise. For each invalid review result it:

1. extracts the missing factual claim from the belief,
2. searches the network for existing `IN` premises,
3. asks an LLM which candidates actually support the missing claim,
4. links those premises into a new justification when not in dry-run mode.

It is essentially a narrower, non-triaging version of `repair_beliefs` focused only on the “search-and-link” repair pattern. It largely duplicates the search-and-link logic in `_do_search_and_link`, but uses different labels/statuses and does not set `repair_action` metadata.

## Contract
**Preconditions**
- `review_results` is an iterable of dict-like review records. Each relevant record should have:
  - `"id"`: the belief ID
  - `"valid"`: falsy when invalid
  - optional `"comment"`: review comment
- `nodes` is a dict mapping belief IDs to node dicts, normally from `export_network()`.
- Node dicts should have `"justifications"` containing dicts with `"antecedents"` lists and optionally `"outlist"` lists.
- `search_fn` must be callable as `search_fn(query, format="json", db_path=db_path)` and return either a JSON string or an iterable of dicts.
- `db_path` points to a database accepted by `api.search` and `api.add_justification` when writes/search are needed.

**Postconditions**
- Returns a list of repair result dicts, one per invalid review result that was iterated.
- Does not mutate `review_results` or `nodes` directly.
- May write new justifications to the database via `api.add_justification`.
- A result with `status == "repaired"` means a match was found and either the write succeeded or `dry_run=True`.
- A result with `status == "error"` means that belief was skipped due to missing node or an exception; `error` should be inspected.

**Important invariant / caveat**
- The write is not transactional. Earlier beliefs can be repaired before a later belief fails.
- In `dry_run=True`, the function still performs extraction, search, and LLM matching, and still reports `"repaired"` when a match is found. It simply skips `api.add_justification`.
- If `r["id"]` is missing, the exception occurs before the per-belief `try`, so the whole function can abort.

## Parameters
- `review_results`
  - List/iterable of review dicts.
  - Filtered with `not r.get("valid", True)`, so missing `"valid"` is treated as valid and skipped. Only falsy `"valid"` values are repaired.
- `nodes`
  - Full network export mapping ID → node dict.
  - Used to find the target belief, format review context, inspect existing antecedents, and check candidate nodes.
- `model="claude"`
  - Passed to `extract_smuggled_claim` and `find_matching_premises`.
- `timeout=300`
  - LLM timeout passed to both LLM calls.
- `db_path=None`
  - Passed to `search_fn` and `api.add_justification`.
- `dry_run=False`
  - When true, no `api.add_justification` call is made, but search/LLM work still runs.
- `search_fn=None`
  - Injectable search implementation for tests or alternate backends.
  - Defaults to:
    ```python
    lambda query, **kw: api.search(query, include_out=True, **kw)
    ```
  - Called as `search_fn(claim, format="json", db_path=db_path)`.

## Return Value
Returns `list[dict]`. Each result has this shape:

```python
{
    "id": belief_id,
    "status": "error" | "extraction_failed" | "no_candidates" | "no_match" | "repaired",
    "smuggled_claim": str | None,
    "matched_premises": list[str],
    "rationale": str | None,
    "error": str | None,
}
```

Typical statuses:
- `"error"` — missing node, malformed data, or exception during processing.
- `"extraction_failed"` — LLM returned no usable smuggled claim.
- `"no_candidates"` — search returned no eligible candidate premises.
- `"no_match"` — candidates existed but LLM matched none.
- `"repaired"` — matched premises were found; DB write attempted unless `dry_run=True`.

The caller must inspect `status` and `error`; a returned result does not mean repair succeeded unless status is `"repaired"`.

## Algorithm
1. Import `.api` lazily.
2. Install default `search_fn` if none was provided.
3. Filter `review_results` to entries where `not r.get("valid", True)`.
4. For each invalid review:
   1. Create an initial result dict with `status="error"`.
   2. Look up the belief node in `nodes`.
      - If missing, set `error="belief not found in network"` and continue.
   3. Format belief context using `format_belief_for_review`.
   4. Extract the review comment.
   5. Call `extract_smuggled_claim(...)`.
   6. If no claim was extracted, set `status="extraction_failed"` and continue.
   7. Build `existing_ants`, the union of antecedents across all existing justifications.
   8. Search using `search_fn(claim, format="json", db_path=db_path)`.
   9. Parse the search result:
      - If string, `json.loads`.
      - Otherwise use as-is.
      - On JSON/type parse failure, treat as empty list.
   10. Build candidates by filtering search results:
       - Skip the target belief itself.
       - Skip IDs already present in any existing antecedent.
       - Require `truth_value == "IN"`.
       - Skip candidates whose node exists and already has justifications.
       - Keep `{"id": ..., "text": ...}`.
   11. If no candidates, set `status="no_candidates"` and continue.
   12. Call `find_matching_premises(claim, candidates, ...)`.
   13. Store rationale and matched IDs.
   14. If no matched IDs, set `status="no_match"` and continue.
   15. If not dry-run:
       - Use the first justification as a template.
       - Copy its antecedents and outlist.
       - Append matched IDs to the antecedents.
       - Call `api.add_justification(...)` with:
         - `sl` = joined original antecedents + matched premises
         - `unless` = joined original outlist
         - label = `"repair-smuggled: <claim[:60]>"`
   16. Set `status="repaired"`.
5. Broadly catch exceptions per belief, store `str(exc)` in `error`, and append the result.

## Side Effects
- Prints progress messages to `sys.stderr`:
  - extracting claim,
  - searching,
  - matching candidates.
- Performs search via `search_fn` / `api.search`.
- Performs up to two LLM calls per repairable belief:
  - `extract_smuggled_claim`
  - `find_matching_premises`
- Calls `api.add_justification` when `dry_run=False` and a match is found.
- Does not update the in-memory `nodes` dict.
- Does not remove or modify the original invalid justification; it relies on `api.add_justification` to persist a new justification.
- Only uses the first justification as the template for copied antecedents/outlist.

## Error Handling
- Per-belief exceptions are caught with `except Exception` and converted into `result["error"]`.
- JSON parsing failure from search is not treated as an error; it becomes an empty search result list.
- Missing node is handled explicitly and produces `status="error"` with a message.
- Some failure modes occur outside the per-belief `try`:
  - `review_results` containing non-dicts can fail during the `r.get(...)` filter.
  - A review record missing `"id"` fails at `r["id"]` before the `try`, aborting the whole call.
- If `api.add_justification` raises, that belief result remains `status="error"` and includes the exception text.
- If the belief has an empty `justifications` list, the `[0]` access can raise `IndexError`, which is caught and returned as an error for that belief.
- Because writes are not transactional, partial repairs may already be committed even if a later belief fails.

## Usage Patterns
Typical use is as a targeted repair pass over review results already known to be invalid:

```python
repairs = repair_smuggled_beliefs(
    review_results=review_results,
    nodes=nodes,
    db_path=db_path,
    dry_run=True,
)
```

Callers should:
- pass only review data whose shape is trusted,
- inspect each result’s `status` and `error`,
- understand that `dry_run=True` is still expensive because search and LLM calls happen,
- not assume `"repaired"` implies validation was rerun or the original justification was removed.

This function is narrower than `repair_beliefs`:
- no triage,
- no `soften`, `abandon`, or `research` handling,
- no `repair_action` metadata,
- status `"repaired"` rather than `"linked"`.

## Dependencies
- Standard library: `json`, `sys`.
- Internal:
  - `.llm.invoke_model` indirectly through `extract_smuggled_claim` / `find_matching_premises`.
  - `.review.format_belief_for_review`.
  - `.api.search` and `.api.add_justification`.
- The function assumes `search_fn` returns either JSON text or an iterable of dicts with `id`, `text`, and `truth_value`.
- It assumes belief IDs are strings suitable for joining with commas.

## Topics to Explore
- [function] `reasons/repair.py:_do_search_and_link` — Compare this helper with `repair_smuggled_beliefs`; they duplicate most search-and-link logic but return different statuses and use different labels.
- [function] `reasons/repair.py:repair_beliefs` — Shows how the same search-and-link pattern fits into the broader triage/soften/abandon/research orchestrator.
- [function] `reasons/api.py:add_justification` — Confirms whether this appends a new justification, replaces one, and how `sl` / `unless` / `label` are persisted.
- [file] `tests/test_repair_premises.py` — Likely exercises candidate filtering, match handling, and dry-run behavior for this function.
- [file] `tests/test_repair.py` — Useful for understanding expected result shapes and error/status contracts across repair routines.

## Beliefs
- `repair-smuggled-beliefs-filters-invalid-reviews` — The function only processes review records whose `valid` field is falsy; missing `valid` defaults to valid and is skipped.
- `repair-smuggled-beliefs-requires-search-fn-json-or-iterable` — `search_fn` must return either a JSON string or an iterable of dicts; parse failures become empty search results rather than errors.
- `repair-smuggled-beliefs-dry-run-still-reports-repaired` — In dry-run mode, the function still performs search and LLM matching and returns `status="repaired"` when a match is found.
- `repair-smuggled-beliefs-write-is-not-transactional` — Repairs are applied per belief through `api.add_justification` with no surrounding transaction, so partial writes are possible.
- `repair-smuggled-beliefs-uses-first-justification-template` — The new justification copies antecedents and outlist from the belief’s first justification, then appends matched premise IDs.
