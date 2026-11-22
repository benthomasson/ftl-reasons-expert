---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: _do_search_and_link in reasons/repair.py

# `_do_search_and_link` — Search-and-Link Repair Pattern

## Purpose

Implements **Pattern 1** of the repair system (`search_and_link`): when a derived belief is flagged invalid because its conclusion smuggles in a premise missing from its antecedents, this function tries to *recover* the missing premise from elsewhere in the network and stitch it into the belief's justification chain.

Two LLM calls are made to identify the missing claim and to select the best candidate premises, and — unless `dry_run` — a new SL justification is written to the database via `api.add_justification`.

## Contract

- **Preconditions**
  - `node` is the node dict for `belief_id` and exists in `nodes`.
  - `node.get("justifications")` is either absent or a list of dicts, each with optional `"antecedents"` (list of str) and `"outlist"` (list of str).
  - `search_fn` accepts `(query, format="json", db_path=...)` and returns either a JSON string or a list of dicts with keys `"id"`, `"text"`, `"truth_value"`.
  - `nodes` contains an entry for every ID returned by `search_fn` *that is not a primitive premise already present in the network* — this is **not** enforced; the code tolerates missing `nodes[sid]` (see below).

- **Postconditions**
  - Returns a 4-tuple `(status, claim, matched_ids, rationale)` where `status ∈ {"extraction_failed", "no_candidates", "no_match", "linked"}`.
  - If `status == "linked"` **and** `not dry_run`, a new justification has been appended to `belief_id` with `sl = original_ants + matched_ids` and `unless = original_outlist`.
  - If `status == "linked"` **and** `dry_run`, no state was mutated.
  - The original justification(s) on `belief_id` are **not** removed; `add_justification` adds a new row (justifications are ORed, so the belief is now derivable via the new, longer chain).

- **Invariants**
  - `matched_ids` is only non-empty when `status == "linked"`.
  - `claim` is always returned, including on failure paths after extraction succeeds.
  - The function never partially applies a repair: either the whole `add_justification` call runs (and may raise, in which case nothing is applied), or it is skipped entirely.

## Parameters

| Parameter | Type | Meaning |
|---|---|---|
| `belief_id` | str | Node ID of the invalid belief to repair. |
| `node` | dict | The belief's node dict from `nodes`. Must contain `"justifications"` (or lack them). |
| `nodes` | dict[str, dict] | Full node snapshot from `export_network()`; used for candidate lookup and to read premise text. |
| `comment` | str | The review finding text — fed to the extraction LLM prompt. |
| `model` | str | LLM model name passed to `extract_smuggled_claim` / `find_matching_premises`. |
| `timeout` | int | LLM HTTP timeout in seconds. |
| `db_path` | path | DB path forwarded to `search_fn` and `add_justification`. |
| `dry_run` | bool | If True, skip the `add_justification` mutation. Does **not** skip LLM/search calls. |
| `search_fn` | callable | Injected query function; default in the orchestrator is `lambda query, **kw: api.search(query, include_out=True, **kw)`. |

Edge cases:
- Missing `justifications` on `node`: `existing_ants` stays empty and `first_just = node.get("justifications", [{}])[0]` yields `{}` → no `original_ants`/`original_outlist`, so `sl` becomes just `matched_ids`.
- A search result whose `sid` is not in `nodes`: `sr_node` is `None`, so the `sr_node and ...` short-circuit **includes** the candidate. This is a latent inconsistency — the DB and the node snapshot are assumed to be in sync.
- Duplicate `matched_ids` from the LLM are not deduped; `parse_match_response` only filters by `valid_ids`.

## Return Value

`(status: str, claim: str, matched_ids: list[str], rationale: str)`

| `status` | Trigger | `claim` | `matched_ids` | `rationale` |
|---|---|---|---|---|
| `"extraction_failed"` | LLM returned falsy claim | `""` | `[]` | `""` |
| `"no_candidates"` | Search + filter yielded nothing | claim | `[]` | `""` |
| `"no_match"` | LLM match returned no IDs | claim | `[]` | match.rationale |
| `"linked"` | Match returned ≥1 ID (mutation applied unless `dry_run`) | claim | matched | match.rationale |

Callers must check `dry_run` themselves to know whether `"linked"` implied a DB mutation — the status does not distinguish.

## Algorithm

1. **Deferred import** of `reasons.api` (avoids a circular import at module load time).
2. Build `belief_context` via `format_belief_for_review(belief_id, nodes)`.
3. **LLM call 1 — extract smuggled claim** (`extract_smuggled_claim`). If empty, bail with `"extraction_failed"`.
4. **Collect existing antecedents** from *all* justifications of the node (union).
5. **Search** with the claim text: `raw = search_fn(claim, format="json", db_path=db_path)`.
6. **Parse search results** — accept a `list` as-is; otherwise `json.loads`. Any `JSONDecodeError`/`TypeError` → `[]`.
7. **Filter candidates** (all must hold):
   - `sid != belief_id` (don't link a belief to itself),
   - `sid not in existing_ants` (no redundant antecedents),
   - `truth_value == "IN"` (node is currently asserted),
   - `sr_node is None or not sr_node.get("justifications")` — **premises only** (leaf nodes, not derived ones).
8. If `candidates` is empty → `"no_candidates"`.
9. **LLM call 2 — match** (`find_matching_premises`). Filters result via `parse_match_response(valid_ids=candidates)`, so spurious IDs are dropped.
10. If `matched_ids` empty → `"no_match"`.
11. **Mutation (unless `dry_run`)**:
    - Read `first_just = node["justifications"][0]` (or `{}`).
    - `new_ants = original_ants + matched_ids`.
    - `api.add_justification(belief_id, sl=",".join(new_ants), unless=",".join(original_outlist), label="repair: linked ...", db_path=db_path)`.
12. Return `("linked", claim, matched_ids, rationale)`.

## Side Effects

- **Two LLM calls** — `extract_smuggled_claim` and `find_matching_premises`; each has network I/O and `timeout`.
- **One search call** — `search_fn(...)` queries the network DB.
- **One DB write** — `api.add_justification(...)` only if `not dry_run` and matches found. This is the only mutation.
- **No in-memory mutation** of `node`, `nodes`, or `candidates` (a fresh `new_ants` list is built).
- **No logging from this function** — all `print(...)` calls live in the orchestrator.

## Error Handling

- **Swallowed**: `json.JSONDecodeError` and `TypeError` during search-result parsing → `search_results = []` → likely `"no_candidates"`.
- **Not swallowed** (propagate to caller `repair_beliefs`, which wraps in `try/except`):
  - Exceptions from `format_belief_for_review`, `extract_smuggled_claim`, `find_matching_premises` (LLM timeouts, network errors).
  - Exceptions from `search_fn`.
  - Exceptions from `api.add_justification` (DB errors, invalid `sl`/`unless` syntax).
- **Partial-success semantics**: if `add_justification` raises, nothing this function did is persisted, but the LLM/search work has already been consumed. The caller will record `status = "error"` and `error = str(exc)`.
- **No explicit return-code for exceptions** — the function assumes the caller's `try/except` handles all failures.

## Usage Patterns

- **Sole caller**: `repair_beliefs` when `triage.pattern == "search_and_link"`.
- **Caller obligations**:
  1. Handle exceptions — this function will happily raise.
  2. After `status == "linked"` **and** `not dry_run`, the caller calls `api.set_metadata(belief_id, "repair_action", "search_and_link", db_path=db_path)`. The function does *not* stamp metadata itself.
  3. Pass a `search_fn` whose results carry `truth_value` and `text` keys, and whose IDs are in `nodes` — otherwise the premise-only filter and the match step may silently misbehave.
- The near-duplicate `repair_smuggled_beliefs` re-implements the same steps inline; both should be kept in sync when behavior changes.

## Dependencies

- `reasons.api` (deferred, circular-import-safe) — `search`, `add_justification`.
- `reasons.review.format_belief_for_review` — builds the belief context for the extraction prompt.
- `reasons.llm.invoke_model` (via `extract_smuggled_claim` / `find_matching_premises`).
- `reasons.repair` locals: `EXTRACT_PROMPT`, `MATCH_PROMPT`, `parse_extract_response`, `parse_match_response`.
- External: `json` only.

## Assumptions Not Enforced by the Type System

- `nodes` is a complete snapshot of the DB; search results not present in `nodes` are treated as legitimate premises.
- All existing justifications share the same "shape" — the code prefers the *first* one as the template, discarding `outlist` values from others.
- `truth_value` in search results uses the string `"IN"` (uppercase, Truth-Maintenance vocabulary). A missing `truth_value` is filtered out.
- `sl` and `unless` strings passed to `api.add_justification` are comma-separated ID lists; comma-containing IDs would break the wire format (no escaping).
- `add_justification` *appends* a new justification (OR semantics) rather than replacing the flagged one.

## Topics to Explore

- [function] `reasons/repair.py:repair_beliefs` — The orchestrator that triages then dispatches to `_do_search_and_link`; shows how `dry_run` and metadata stamping interact with this function's return.
- [function] `reasons/api.py:add_justification` — Confirms whether SL justifications are appended (ORed) and how `unless` (outlist) is stored; validate the "premise-only" candidate filter interacts correctly with multiple justifications.
- [function] `reasons/api.py:search` — What `search_fn` actually returns (fields, `truth_value`, pagination, `include_out`), and whether results are guaranteed to be present in the `nodes` snapshot.
- [file] `tests/test_repair.py` — Exercises the four statuses of `_do_search_and_link`; a good proxy for the implicit contract.
- [general] `repair-pattern-dry-run-semantics` — Why `dry_run` returns `"linked"` even though no mutation occurred, and where the caller is expected to reconcile this.

## Beliefs

- `search-and-link-returns-four-tuple` — `_do_search_and_link` returns `(status, claim, matched_ids, rationale)` where status is one of `extraction_failed`, `no_candidates`, `no_match`, or `linked`.
- `search-and-link-filters-to-leaf-in-premises` — Candidates must have `truth_value == "IN"`, differ from `belief_id`, not appear in `existing_ants`, and have no `justifications` of their own (premise-only filter).
- `search-and-link-uses-first-justification-only` — When `node["justifications"]` contains multiple entries, only `justifications[0]` supplies `original_ants` and `original_outlist` for the new SL justification.
- `search-and-link-dry-run-skips-db-mutation` — With `dry_run=True` the function still performs both LLM calls and search but skips `api.add_justification`, yet still returns `status == "linked"` on a successful match.
- `search-and-link-json-decode-errors-yield-empty-results` — Malformed JSON from `search_fn` is caught and converted to `[]`, producing `"no_candidates"` rather than raising.
