---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/repair.py

# `reasons/repair.py` Explanation

## Purpose

`reasons/repair.py` is the repair subsystem for invalid beliefs in the Truth Maintenance System. It consumes review results where a belief has been flagged invalid, triages the belief into a repair strategy, and then applies that strategy through the `reasons.api` mutation layer.

It implements four logical repair patterns:

1. **`search_and_link`** — extract the missing “smuggled” premise, search the network for candidate premises, match them via LLM, and add a new justification linking them.
2. **`soften`** — rewrite an overstated belief so it follows from its antecedents, then supersede the old belief with the softened text.
3. **`abandon`** — retract a belief whose dependency chain is too broken to repair.
4. **`research`** — leave the belief intact but tag it as needing further investigation.

There are two public orchestration entry points:

- `repair_beliefs(...)` — the general triage-driven orchestrator.
- `repair_smuggled_beliefs(...)` — a specialized search-and-link-only implementation.
- `research_beliefs = repair_beliefs` — backward-compatible alias.

The module owns prompt construction, LLM invocation, tolerant parsing of LLM responses, candidate filtering, depth calculation, metadata tagging, and per-belief error capture.

---

## Key Components

### Prompt Constants

- `EXTRACT_PROMPT`  
  Asks the LLM to identify the specific factual claim the belief conclusion assumes but its antecedents do not provide. Expects one sentence.

- `MATCH_PROMPT`  
  Asks the LLM to match a smuggled claim against candidate premises and return JSON:
  ```json
  {"matched_ids": ["id-1"], "rationale": "..."}
  ```
  Explicitly warns against topically related but non-supporting premises.

- `TRIAGE_PROMPT`  
  Asks the LLM to choose one of:
  - `search_and_link`
  - `soften`
  - `abandon`
  - `research`

  Includes `depth` and `flagged_ancestors` as dependency context. Returns:
  ```json
  {"pattern": "...", "rationale": "..."}
  ```

- `SOFTEN_PROMPT`  
  Asks the LLM to rewrite an overstated belief so it follows strictly from its antecedents. Returns:
  ```json
  {"softened_text": "...", "rationale": "..."}
  ```

- `VALID_PATTERNS = {"search_and_link", "soften", "abandon", "research"}`  
  Used by `parse_triage_response` to reject invalid triage outputs.

### Response Parsers

- `parse_extract_response(response) -> str`  
  Strips leading `"Smuggled claim:"` prefixes and matching single/double quotes. Returns `""` for empty responses.

- `parse_match_response(response, valid_ids) -> dict`  
  Scans the response for JSON using `json.JSONDecoder.raw_decode`, starting at every `{`. Filters `matched_ids` to only IDs in `valid_ids`. Returns:
  ```python
  {"matched_ids": [...], "rationale": "..."}
  ```
  Falls back to empty matches if no valid JSON object is found.

- `parse_triage_response(response) -> dict`  
  Similar raw-JSON scanning, but only accepts a dict whose `pattern` is in `VALID_PATTERNS`. Invalid pattern objects are skipped. Returns `{"pattern": "", "rationale": ""}` if nothing valid is found.

- `parse_soften_response(response) -> dict`  
  Scans for a JSON object with a non-empty `softened_text`. Returns:
  ```python
  {"softened_text": "...", "rationale": "..."}
  ```

### LLM Wrappers

- `extract_smuggled_claim(belief_context, review_comment, model="claude", timeout=300) -> str`  
  Formats `EXTRACT_PROMPT`, calls `invoke_model`, and parses the result.

- `find_matching_premises(smuggled_claim, candidates, model="claude", timeout=300) -> dict`  
  If `candidates` is empty, returns:
  ```python
  {"matched_ids": [], "rationale": "no candidates found"}
  ```
  Otherwise formats candidates as:
  ```text
  - `id`: text
  ```
  Calls `invoke_model`, then `parse_match_response`.

- `triage_belief(belief_context, review_comment, depth=0, flagged_ancestors=0, model="claude", timeout=300) -> dict`  
  Formats `TRIAGE_PROMPT`, calls `invoke_model`, then `parse_triage_response`.

- `soften_belief(belief_text, antecedents_text, model="claude", timeout=300) -> dict`  
  Formats `SOFTEN_PROMPT`, calls `invoke_model`, then `parse_soften_response`.

### Network Helpers

- `_compute_depth(node_id, nodes, memo=None) -> int`  
  Computes derivation depth:
  - No node or no justifications → `0`
  - Otherwise → `max(antecedent_depth) + 1`

  Uses `memo` to avoid repeated work and to guard cycles. It sets `memo[node_id] = 0` before recursing.

- `_format_antecedents(node, nodes) -> str`  
  Produces `- antecedent_id: antecedent_text` lines for all antecedents across all justifications. Returns `"(no antecedents)"` if none exist.

- `_do_search_and_link(...) -> (status, claim, matched_ids, rationale)`  
  Implements the search-and-link strategy for a single belief. It:
  1. Formats the belief context.
  2. Extracts the smuggled claim.
  3. Collects existing antecedents.
  4. Searches via `search_fn(claim, format="json", db_path=db_path)`.
  5. Filters search results to valid candidate premises.
  6. Calls `find_matching_premises`.
  7. If not `dry_run`, calls `api.add_justification(...)` with a new antecedent set.

### Orchestrators

- `repair_beliefs(review_results, nodes, model="claude", timeout=300, db_path=None, dry_run=False, search_fn=None) -> list[dict]`  
  General orchestrator. Filters review results to invalid beliefs, triages each one, dispatches to the selected pattern, applies mutations unless `dry_run=True`, and returns one result dict per invalid belief.

  Result dict fields:
  - `id`
  - `pattern`
  - `status`
  - `rationale`
  - `smuggled_claim`
  - `matched_premises`
  - `softened_text`
  - `error`
  - optionally `new_id` after softening.

- `repair_smuggled_beliefs(review_results, nodes, model="claude", timeout=300, db_path=None, dry_run=False, search_fn=None) -> list[dict]`  
  Specialized search-and-link-only orchestrator. It does not triage and does not set `repair_action` metadata. It returns:
  - `id`
  - `status`
  - `smuggled_claim`
  - `matched_premises`
  - `rationale`
  - `error`

  Successful status is `"repaired"`, not `"linked"`.

- `research_beliefs = repair_beliefs`  
  Alias likely kept for API compatibility or older callers.

---

## Patterns

### 1. LLM-as-triage + strategy dispatch

`repair_beliefs` acts as an orchestrator. It first calls `triage_belief`, then dispatches to one of four branches. This is effectively a strategy pattern where the strategy is selected by an LLM.

The actual mutation responsibilities are delegated to `reasons.api`:

- `api.add_justification(...)`
- `api.supersede_with_text(...)`
- `api.retract_node(...)`
- `api.set_metadata(...)`
- `api.search(...)`

### 2. Tolerant JSON parsing

LLM responses are not assumed to be pure JSON. All JSON parsers use `json.JSONDecoder().raw_decode(...)` while scanning for `{` characters. This lets the module accept prose-wrapped JSON, code fences, or extra explanation.

### 3. Dependency injection for search

Both orchestrators accept an injectable `search_fn`. The default is:

```python
lambda query, **kw: api.search(query, include_out=True, **kw)
```

Tests can replace this to avoid touching the real database.

### 4. `dry_run` mutation control

`dry_run=True` suppresses database mutations but does not suppress LLM calls or search calls. The module still triages, extracts, searches, matches, and reports intended actions.

### 5. Lazy import of `api`

`from . import api` is done inside functions rather than at module import time. This avoids circular-import problems because `reasons/api.py` imports or exposes repair functionality.

### 6. Metadata audit trail

Successful non-dry-run repairs write metadata:

- Search-and-link: `repair_action = "search_and_link"`
- Soften: `repair_action = "softened"` on the new superseding node
- Abandon: `repair_action = "abandoned"`
- Research: `repair_research = <rationale>` and `repair_action = "research"`

---

## Dependencies

### Imports

- `json` — parsing search results and LLM JSON responses.
- `sys` — progress messages to `stderr`.
- `from .llm import invoke_model` — generic LLM invocation.
- `from .review import format_belief_for_review` — builds the structured belief context used in prompts.

### Lazy Imports

Within functions:

```python
from . import api
```

Used for:
- `api.search(...)`
- `api.add_justification(...)`
- `api.supersede_with_text(...)`
- `api.retract_node(...)`
- `api.set_metadata(...)`

### Imported By

The module is imported by:

- `reasons/api.py`
- `reasons_lib/repair.py`
- `reasons_lib/repair_premises.py`
- `tests/test_repair.py`
- `tests/test_repair_premises.py`
- `tests/test_research.py`

The repository contains a parallel `reasons_lib` tree. If `reasons/repair.py` and `reasons_lib/repair.py` are intended to stay in sync, changes should be checked in both places.

---

## Flow

### General flow: `repair_beliefs`

1. Filter `review_results`:
   ```python
   invalid = [r for r in review_results if not r.get("valid", True)]
   ```
   Missing `valid` is treated as valid.

2. Build `invalid_ids` for ancestor counting.

3. For each invalid belief:
   1. Look up the node in `nodes`. If missing, record error.
   2. Format belief context via `format_belief_for_review`.
   3. Compute depth via `_compute_depth`.
   4. Count flagged ancestors by checking antecedents against `invalid_ids`.
   5. Call `triage_belief(...)`.
   6. If triage returns no pattern, status becomes `"triage_failed"`.
   7. Dispatch based on pattern:

      - **`search_and_link`**
        - Call `_do_search_and_link`.
        - Statuses can be:
          - `"extraction_failed"`
          - `"no_candidates"`
          - `"no_match"`
          - `"linked"`
        - If `status == "linked"` and not dry-run, set `repair_action` metadata.

      - **`soften`**
        - Format antecedents.
        - Call `soften_belief`.
        - If empty softened text → `"soften_failed"`.
        - If not dry-run:
          - `api.supersede_with_text(...)`
          - Store `new_id`
          - Set `repair_action = "softened"` on the new node.
        - Status becomes `"softened"`.

      - **`abandon`**
        - If not dry-run:
          - `api.retract_node(...)`
          - Set `repair_action = "abandoned"`.
        - Status becomes `"abandoned"`.

      - **`research`**
        - If not dry-run:
          - Set `repair_research` metadata to the triage rationale.
          - Set `repair_action = "research"`.
        - Status becomes `"needs_research"`.

4. Return all result dicts.

### Specialized flow: `repair_smuggled_beliefs`

1. Filter invalid review results.
2. For each invalid belief:
   1. Look up node.
   2. Format belief context.
   3. Extract smuggled claim.
   4. If extraction fails → `"extraction_failed"`.
   5. Collect existing antecedents.
   6. Search network.
   7. Filter candidates.
   8. Match claim against candidates.
   9. If no candidates → `"no_candidates"`.
   10. If no match → `"no_match"`.
   11. If not dry-run, call `api.add_justification(...)`.
   12. Status becomes `"repaired"`.

This path is essentially the search-and-link half of the general orchestrator, without triage, metadata, soften, abandon, or research.

---

## Invariants

- **Only explicitly invalid beliefs are processed.**  
  `not r.get("valid", True)` means missing `valid` is treated as valid.

- **Search-and-link candidates are restricted to premises.**  
  Candidate search results are rejected if:
  - `sid == belief_id`
  - `sid in existing_ants`
  - `sr.get("truth_value") != "IN"`
  - `nodes.get(sid)` has a non-empty `justifications` list

  This means only `IN` premises are considered, not derived beliefs.

- **Hallucinated match IDs are discarded.**  
  `parse_match_response` filters LLM-returned IDs against the actual candidate ID set.

- **Depth treats premises as depth 0.**  
  Nodes with no justifications have depth `0`; nodes with justifications have depth `max(antecedent_depth) + 1`.

- **Depth memoization is shared across the whole repair run.**  
  `_compute_depth` uses a `depth_memo` reused for all invalid beliefs. Mutations performed earlier in the same repair batch are not reflected in cached depths.

- **Mutations require `not dry_run`.**  
  LLM and search calls still happen during dry runs, but `add_justification`, `supersede_with_text`, `retract_node`, and `set_metadata` are skipped.

- **Search-and-link bases the new justification on the first existing justification.**  
  `_do_search_and_link` and `repair_smuggled_beliefs` read `node.get("justifications", [{}])[0]` and reuse its `antecedents` and `outlist` as the base for the new justification. Other justifications are not directly modified.

- **Successful triage patterns must be one of the four known strings.**  
  `parse_triage_response` rejects any pattern not in `VALID_PATTERNS`.

- **`repair_smuggled_beliefs` does not write repair metadata.**  
  Unlike `repair_beliefs`, it does not call `api.set_metadata`.

---

## Error Handling

- **LLM parse failures are generally swallowed.**  
  Parsers return empty strings or empty match lists rather than raising. This turns parse failures into statuses such as:
  - `"extraction_failed"`
  - `"triage_failed"`
  - `"soften_failed"`
  - `"no_match"`

- **Search result parse failures become empty lists.**  
  In both orchestrators:
  ```python
  try:
      search_results = json.loads(raw) if isinstance(raw, str) else raw
  except (json.JSONDecodeError, TypeError):
      search_results = []
  ```
  This usually leads to `"no_candidates"` rather than an exception.

- **Missing belief nodes are reported per belief.**  
  If `nodes.get(belief_id)` is missing, the result gets:
  ```python
  result["error"] = "belief not found in network"
  ```

- **Per-belief exceptions are captured and do not abort the batch.**  
  `repair_beliefs` and `repair_smuggled_beliefs` wrap each belief in `try/except Exception as exc` and set `result["error"] = str(exc)`. The loop continues to the next invalid belief.

- **Mutations are not transactional.**  
  If `api.add_justification(...)` succeeds but a later metadata write fails, the network change may already be committed while the result reports an error. Similarly, if `supersede_with_text` succeeds and `set_metadata` fails, the new node exists but the result may be marked as `"error"`.

- **Status/error consistency is branch-dependent.**  
  In the search-and-link branch, `result["status"] = status` happens before `api.set_metadata(...)`. If metadata writing fails, the status can remain `"linked"` while `error` is also set. In the soften, abandon, and research branches, status is set after mutation/metadata, so an exception can leave the status as `"error"` even if an earlier mutation succeeded.

---

## Topics to Explore

- [file] `reasons/review.py` — Defines `format_belief_for_review` and the review-result schema consumed by this module.
- [file] `reasons/api.py` — Exposes repair functions and implements the mutation primitives named `add_justification`, `supersede_with_text`, `retract_node`, `set_metadata`, and `search`.
- [function] `reasons/repair.py:_compute_depth` — Contains the cycle-guarded depth calculation and shared memo semantics used by triage.
- [file] `tests/test_repair.py` — Likely exercises triage dispatch, statuses, search-and-link, soften, abandon, research, and dry-run behavior.
- [general] `repair-patterns` — Useful to compare the four repair strategies and how their statuses and metadata differ.

## Beliefs

- `repair-beliefs-only-processes-valid-false` — `repair_beliefs` and `repair_smuggled_beliefs` process only review results where `valid` is explicitly false; missing `valid` defaults to valid.
- `search-and-link-restricts-candidates-to-in-premises` — Search-and-link candidate selection excludes the belief itself, existing antecedents, non-`IN` nodes, and nodes with justifications.
- `repair-beliefs-triages-with-llm-before-mutation` — `repair_beliefs` calls `triage_belief` first and only executes a repair branch when triage returns a pattern in `VALID_PATTERNS`.
- `dry-run-suppresses-api-mutations` — `dry_run=True` still performs LLM and search calls but skips `add_justification`, `supersede_with_text`, `retract_node`, and `set_metadata`.
- `json-parsers-scan-with-raw-decode` — The LLM response parsers use `json.JSONDecoder.raw_decode` while scanning each `{` so they tolerate prose around JSON.
