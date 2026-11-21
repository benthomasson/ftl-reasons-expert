---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: repair_beliefs in reasons/repair.py

## `repair_beliefs` — Orchestrator for belief repair

### 1. Purpose

This is the top-level entry point for the "repair" phase of a Truth Maintenance System. Given the output of a review pass (`review_results`) that marks each belief as valid or invalid, and the full belief network (`nodes`), it repairs every invalid belief by:

1. Asking an LLM to **triage** each invalid belief into one of four repair patterns (`search_and_link`, `soften`, `abandon`, `research`).
2. Dispatching to the corresponding repair implementation.
3. Returning a structured record of what happened for each belief.

It exists so that a single CLI/API call can drive the whole repair loop and report a summary, instead of the caller having to filter invalid beliefs, do triage, and route to four different helpers itself.

### 2. Contract

**Preconditions (assumed, not enforced by types):**
- `review_results` is a list of dicts; each has at least `"id"`, may have `"valid"` and `"comment"`. Missing `"valid"` is treated as `True` (valid) — so those entries are skipped.
- `nodes` is a dict keyed by belief id whose values support `.get("justifications", ...)`, and each justification has `"antecedents"` and (for linking) `"outlist"`.
- For any node whose repair pattern needs antecedents (`soften`), `nodes` contains those antecedent nodes.
- `search_fn(claim, format="json", db_path=...)` returns a JSON string or a list of `{"id", "text", "truth_value"}` dicts.
- `reasons.api` is importable and exposes `search`, `set_metadata`, `supersede_with_text`, `retract_node`, `add_justification`.

**Postconditions:**
- Returns exactly one result dict per entry in `invalid` (i.e., per `review_results` entry with `valid` falsy or missing-true).
- All exceptions raised while processing one belief are caught and recorded in that belief's `result["error"]`; the loop continues to the next belief.
- If `dry_run=True`, no mutation API is called (`set_metadata`, `supersede_with_text`, `retract_node`, `add_justification`); `search` is still called because it is read-only and needed for triage → link.

**Invariants:**
- Every result dict starts with `status="error"`; only successful dispatch overrides it.
- `result["pattern"]` is set from triage even on triage failure (empty string, not `None`).
- Metadata write timing: for `search_and_link`, metadata is only written when `status == "linked"` **and** `not dry_run`. For `soften`, after a successful `supersede_with_text`. For `abandon`/`research`, only under `not dry_run`.

### 3. Parameters

| Name | Type | Meaning / notes |
|---|---|---|
| `review_results` | `list[dict]` | Rows from a review pass. `r.get("valid", True)` is used to decide invalidity. Missing `"valid"` ⇒ considered valid ⇒ skipped. `"comment"` is passed verbatim to triage as the review finding. |
| `nodes` | `dict[str, dict]` | Full belief network export. Each node may have `"text"` and `"justifications"` (list of `{"antecedents": [...], "outlist": [...]}`). |
| `model` | `str` | LLM identifier forwarded to `invoke_model` via triage/extract/match/soften. |
| `timeout` | `int` | Per-LLM-call timeout in seconds; passed to every LLM helper. |
| `db_path` | `str \| None` | Database path used by all `api.*` calls and by `search_fn` (via `**kw`). |
| `dry_run` | `bool` | If `True`, do not persist any repair (still performs read-only triage/extract/match/soften) so you can preview outcomes. |
| `search_fn` | `callable \| None` | Injection point for search. Defaults to a lambda wrapping `api.search(query, include_out=True, **kw)`, so the callable must accept `format=...` and `db_path=...` keywords and return JSON-ish results. |

**Edge cases:**
- Rows whose `id` isn't in `nodes` produce a result with `error="belief not found in network"`.
- Rows with no `"comment"` get `""` for triage.
- `search_fn=None` is only latched **after** the `api` import inside this function; tests can pass a stub to avoid DB access.

### 4. Return Value

A `list[dict]` with one entry per invalid belief, each having:

- `id`: belief id (mirrored from `review_results`).
- `pattern`: triage result (`"search_and_link"`, `"soften"`, `"abandon"`, `"research"`, or `""` on triage failure).
- `status`: one of:
  - `"linked"`, `"softened"`, `"abandoned"`, `"needs_research"` — success.
  - `"triage_failed"` — LLM returned no valid pattern.
  - `"soften_failed"` — soften LLM returned empty text.
  - `"extraction_failed"`, `"no_candidates"`, `"no_match"` — from `_do_search_and_link`.
  - `"error"` (default) — exception or missing node.
- `rationale`: string from triage, overwritten by pattern-specific rationale where applicable.
- `smuggled_claim`: populated for `search_and_link` (may be `None` for other patterns).
- `matched_premises`: list of matched premise ids (search_and_link only).
- `softened_text`: rewritten claim (soften only).
- `error`: exception message string when applicable, otherwise `None`.
- `new_id`: only present for successful soften — the id of the superseding node.

Callers must handle statuses themselves (e.g., map `error`/`*_failed` to non-zero exit codes or to a follow-up queue) and must tolerate `new_id` being absent.

### 5. Algorithm

1. **Late-import `api`** (deferred so tests can stub `reasons.api` after import).
2. **Default `search_fn`** to `api.search(query, include_out=True, **kw)`.
3. **Filter invalid** rows: `[r for r in review_results if not r.get("valid", True)]`.
4. **Build `invalid_ids`** set (used for ancestor weighting).
5. **Create one shared `depth_memo`** across all iterations — so depth is computed once per node per call, not once per belief.
6. **For each invalid row `r`:**
   - Initialize a fixed-shape result dict with `status="error"`.
   - Look up `node = nodes.get(belief_id)`; if missing, set `error` and continue.
   - `belief_context = format_belief_for_review(...)` — a text snapshot of text + justifications for LLM prompts.
   - `depth = _compute_depth(belief_id, nodes, depth_memo)` — 0 for premises, `max(antecedent depths)+1` otherwise.
   - **Count flagged ancestors**: iterate direct justifications → antecedents; increment if `ant_id in invalid_ids`. **Note: this is direct (one hop) only, not transitive.**
   - `triage = triage_belief(...)` — one LLM call. Populate `pattern` and `rationale`. If no pattern, set `status="triage_failed"` and continue.
   - **Dispatch on pattern:**
     - **`search_and_link`**: call `_do_search_and_link(...)` which returns `(status, claim, matched_ids, rationale)`. Assign those. If `status == "linked"` and not dry run, write `repair_action=search_and_link` metadata.
     - **`soften`**: build antecedent text, call `soften_belief`. If empty result → `soften_failed`. Else store text and rationale; if not dry run, `supersede_with_text` and record `new_id`, plus `repair_action=softened` on the new node.
     - **`abandon`**: if not dry run, `retract_node(reason=...)` then set `repair_action=abandoned` metadata. Status `"abandoned"`.
     - **`research`**: if not dry run, write `repair_research=<rationale>` then `repair_action=research`. Status `"needs_research"`.
   - Wrap the whole per-belief body in `try/except Exception` and store `str(exc)` in `result["error"]`.
7. Append result and move on.

### 6. Side Effects

- **stderr logging**: `print(..., file=sys.stderr)` for triage start and pattern dispatch.
- **Read-only I/O via `search_fn`** (default → `api.search(..., include_out=True)`), which queries the belief DB.
- **Mutating I/O via `reasons.api`**, only when `dry_run=False`:
  - `api.set_metadata(id, "repair_action", <pattern>, db_path=...)` — always on success.
  - `api.set_metadata(id, "repair_research", rationale, db_path=...)` — research only.
  - `api.add_justification(id, sl=..., unless=..., ...)` — inside `_do_search_and_link` (linking).
  - `api.supersede_with_text(id, softened, ...)` — soften.
  - `api.retract_node(id, reason=..., ...)` — abandon.
- **LLM calls**: one triage call per invalid belief; plus extract + match for search_and_link, or soften for soften.
- **No shared state mutation** other than the local `depth_memo` cache (which does go stale if callers mutate `nodes` mid-loop — the code assumes `nodes` is immutable for the call).

### 7. Error Handling

- `_compute_depth`'s memo cycle guard sets `memo[node_id] = 0` before recursing, so cycles won't infinite-loop (but the resulting depths on cyclic subgraphs are approximate/incorrect — this is a silent correctness assumption: real belief graphs are DAGs).
- Any exception from triage, pattern helpers, LLM calls, or `api.*` mutations inside the per-belief `try` is swallowed into `result["error"]` and the loop continues.
- The outer `api` import is not wrapped — an import error would propagate.
- The `format_belief_for_review` call is *outside* its own try; if it raises, it's caught by the same per-belief try (fine).
- A missing `search_fn` return that isn't JSON-parseable is handled inside `_do_search_and_link` (`search_results = []`); the caller just sees `no_candidates`.
- `_do_search_and_link` may call `node.get("justifications", [{}])[0]` — **assumes at least one justification exists**. If the node has no justifications, `[0]` raises `IndexError`, caught by the outer try, resulting in `status="error"` with the exception message. This is a real edge case.

### 8. Usage Patterns

Typical caller (CLI or another pipeline stage):

```python
from reasons.review import review_beliefs  # produces review_results
from reasons.repair import repair_beliefs

review_results = review_beliefs(nodes, model="claude")
repairs = repair_beliefs(review_results, nodes, model="claude", db_path=path)
for r in repairs:
    if r["status"] in {"linked", "softened", "abandoned", "needs_research"}:
        ...
    else:
        log_failure(r)
```

Obligations on the caller:
- **Re-export `nodes`** after repairs if you plan to review again (mutated state lives in DB).
- **Treat `new_id` for soften** as the live id of the softened belief; the original was superseded.
- **Use `dry_run=True`** first if you want the LLM's proposed claims/softenings without persisting.
- **Provide a `search_fn` in tests** to avoid hitting `api.search`.
- Note: `research_beliefs = repair_beliefs` is an alias — this function satisfies both "research" and "repair" call-sites.

### 9. Dependencies

- **Internal:**
  - `reasons.api` (late-imported; `search`, `set_metadata`, `supersede_with_text`, `retract_node`, `add_justification`).
  - `reasons.review.format_belief_for_review` — turns a node + network into prompt text.
  - Same-module helpers: `triage_belief`, `_compute_depth`, `_format_antecedents`, `soften_belief`, `_do_search_and_link`, `parse_*` functions used indirectly.
  - `reasons.llm.invoke_model` (via the helpers) for all LLM calls.
- **Stdlib:** `json`, `sys`.

---

## Topics to Explore

- [file] `reasons/repair.py` — Companion helper `_do_search_and_link` and the legacy `repair_smuggled_beliefs` show a parallel, single-pattern implementation; useful for diffing scope and status vocabularies.
- [function] `reasons/review.py:format_belief_for_review` — Determines exactly what the LLM sees during triage; a change here reshapes the repair behavior in subtle ways.
- [function] `reasons/repair.py:_compute_depth` — Memoized DFS over justifications; relevant to cyclic-dependency assumptions and the `flagged_ancestors` scoring heuristic.
- [file] `tests/test_repair.py` — Shows the injected `search_fn` contract and which statuses are asserted on, including dry-run behavior.
- [general] `truth-maintenance-repair-workflow` — How review → triage → repair loops are invoked from the CLI and what state they persist in the DB.

## Beliefs

- `repair-briefs-filters-invalid-by-valid-field` — `repair_briefs` skips any `review_results` entry where `valid` is truthy or absent; only entries with `valid == False` (or falsy values) are processed.
- `repair-briefs-never-propagates-per-belief-exceptions` — Any exception raised while handling a single invalid belief is caught and stored in that belief's `result["error"]`; the loop continues.
- `dry-run-prevents-api-mutations` — When `dry_run=True`, `api.set_metadata`, `supersede_with_text`, `retract_node`, and `add_justification` are never called, but `search_fn` is still invoked because search is read-only.
- `flagged-ancestors-counts-only-direct-antecedents` — `flagged_ancestors` counts one-hop antecedent ids present in `invalid_ids`; it does not walk the transitive dependency tree.
- `search-and-link-metadata-written-only-on-linked-status` — `api.set_metadata(..., "repair_action", "search_and_link")` runs only when `status == "linked"` and `not dry_run`, so `extraction_failed`/`no_candidates`/`no_match` leave the belief untouched.
