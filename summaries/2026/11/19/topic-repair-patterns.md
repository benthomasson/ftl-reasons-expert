---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Useful to compare the four repair strategies and how their statuses and metadata differ.

# Comparing the four repair strategies

The four patterns are enumerated in `VALID_PATTERNS` in `reasons/repair.py`:

```python
VALID_PATTERNS = {"search_and_link", "soften", "abandon", "research"}
```

`triage_belief` (via `TRIAGE_PROMPT`) picks one per invalid belief, and `repair_beliefs` (`reasons/repair.py`) dispatches into a branch for each. The result dict is pre-seeded with the same shape for every belief:

```python
result = {
    "id": belief_id, "pattern": None, "status": "error",
    "rationale": None, "smuggled_claim": None,
    "matched_premises": [], "softened_text": None, "error": None,
}
```

So the fields an individual pattern fills in — and the `status` value it reports — are the main way to distinguish outcomes. Below each strategy is broken down against `search_and_link`, `soften`, `abandon`, and `research`.

---

## 1. `search_and_link` — remove the smuggled premise by linking an existing one

Handled in `_do_search_and_link` and the matching branch of `repair_beliefs`.

**Extra LLM calls beyond triage:** two (`extract_smuggled_claim` → `find_matching_premises`).

**Statuses produced (in `result["status"]`):**

| Status | Where set |
|---|---|
| `extraction_failed` | `_do_search_and_link` returns `"extraction_failed"` when `parse_extract_response` yields an empty claim |
| `no_candidates` | returned when search results are all filtered out (self, existing antecedents, non-`IN`, or derived nodes with `justifications`) |
| `no_match` | returned when `find_matching_premises` returns `matched_ids == []` |
| `linked` | returned after a successful `api.add_justification` call (or in dry-run when status would be `linked`) |

**Extra fields populated on `result`:**
- `smuggled_claim` — the string from `extract_smuggled_claim`
- `matched_premises` — the filtered `matched_ids` list

**Structural effect:** calls `api.add_justification(belief_id, sl=..., unless=..., label=...)` on the **existing** node, preserving the original `outlist` in `unless`. The node itself is not replaced.

**Metadata written:** `api.set_metadata(belief_id, "repair_action", "search_and_link")` — only when `not dry_run and status == "linked"`.

**Rationale precedence:** the triage rationale is overwritten with the LLM match rationale whenever the match returns a non-empty rationale (`if rationale: result["rationale"] = rationale`), so downstream consumers may see a different rationale than what triage had.

---

## 2. `soften` — weaken the claim to match antecedents

Handled in `soften_belief` + the matching branch.

**Extra LLM calls beyond triage:** one.

**Statuses produced:**

| Status | Where set |
|---|---|
| `soften_failed` | returned early when `parse_soften_response` yields an empty `softened_text` |
| `softened` | set after building the new node (or in dry-run) |

**Extra fields populated on `result`:**
- `softened_text` — the rewritten claim
- `new_id` — assigned only when `not dry_run` (the ID of the superseding node)

**Structural effect:** `api.supersede_with_text(belief_id, softened, db_path=...)` — this **creates a new belief** and supersedes the old one, unlike every other pattern.

**Metadata written:** `api.set_metadata(sup["new_id"], "repair_action", "softened")` — note the metadata goes on the **new** node, not the original `belief_id`.

**Rationale:** always overwritten with the `soften_belief` rationale.

---

## 3. `abandon` — retract the belief

Handled entirely inside the `elif pattern == "abandon"` branch.

**Extra LLM calls beyond triage:** none.

**Statuses produced:**

| Status | Notes |
|---|---|
| `abandoned` | set unconditionally after the branch runs (dry-run too) |

**Extra fields on `result`:** none beyond the common fields.

**Structural effect:** `api.retract_node(belief_id, reason=f"repair: abandoned — {triage.get('rationale', '')}", db_path=db_path)` — retracts the **original** node.

**Metadata written:** `api.set_metadata(belief_id, "repair_action", "abandoned")` on the original belief.

**Rationale:** remains the triage rationale — but it is also embedded into the `retract_node` `reason` string, so it may be duplicated between the metadata store and the retraction reason.

---

## 4. `research` — park the belief pending external investigation

Handled inside the `elif pattern == "research"` branch. This is a **metadata-only** outcome — no text is changed, nothing is retracted, and no new node is created.

**Extra LLM calls beyond triage:** none.

**Statuses produced:**

| Status | Notes |
|---|---|
| `needs_research` | set after the branch runs (dry-run too) |

**Extra fields on `result`:** none.

**Structural effect:** none — the belief stays as-is (still likely `IN` and still flagged).

**Metadata written:** two keys, unlike any other pattern —
1. `api.set_metadata(belief_id, "repair_research", triage.get("rationale", ""))`
2. `api.set_metadata(belief_id, "repair_action", "research")`

So `research` is the only pattern that stores a free-form rationale as metadata.

---

## Side-by-side summary

| Pattern | Extra LLM calls | Success status | Other statuses | Result extras | Structural effect | Metadata key(s) on which node |
|---|---|---|---|---|---|---|
| `search_and_link` | 2 | `linked` | `extraction_failed`, `no_candidates`, `no_match` | `smuggled_claim`, `matched_premises` | adds a justification to the **same** node | `repair_action=search_and_link` on `belief_id` |
| `soften` | 1 | `softened` | `soften_failed` | `softened_text`, `new_id` | `supersede_with_text` → **new** node, old superseded | `repair_action=softened` on `new_id` |
| `abandon` | 0 | `abandoned` | — | — | `retract_node` | `repair_action=abandoned` on `belief_id` |
| `research` | 0 | `needs_research` | — | — | none | `repair_action=research` **and** `repair_research=<rationale>` on `belief_id` |

**Shared, non-pattern statuses** (from the outer loop, not any pattern branch):
- `error` — default; set explicitly when `belief_id` is missing from `nodes` (`"belief not found in network"`), or via the `try/except` when any exception is raised.
- `triage_failed` — when `parse_triage_response` cannot produce a valid pattern.

**`dry_run` behaviour is uniform in the `status` it returns** but not in effects: every pattern still reports its success status (`linked`, `softened`, `abandoned`, `needs_research`), while all writes (`add_justification`, `supersede_with_text`, `retract_node`, `set_metadata`) are gated behind `if not dry_run`. For `search_and_link`, `dry_run` short-circuits the `try/except` where `_do_search_and_link` skips `add_justification`, but the branch still falls through to `if not dry_run and status == "linked"` for the metadata write — so dry-run reports `linked` but leaves both the network and the metadata untouched.

---

## Observation gaps

Two tool results returned no matches and appear not to have run successfully against the source (patterns like `status`/`metadata` and `strategy`/`STRATEG` should have matched a lot in `reasons/repair.py` and `reasons/cli.py`):
- `find_repair_strategy_definitions` — 0 matches
- `find_status_metadata_references` — 0 matches
- `find_repair_usages_in_cli` — 0 matches

The `repair.py` content was also truncated at 500/618 lines and cut off mid-docstring of `repair_smuggled_beliefs`, so the older `repair_smuggled_beliefs` orchestrator's exact status set (`repaired` in `tests/test_repair.py`, distinct from `linked` in `repair_beliefs`) is only visible via the tests. Also note `research_beliefs = repair_beliefs` — the "research" orchestrator is just an alias of the general repair orchestrator, implying `research` was added after `repair_beliefs` was already generalised; the tests in `tests/test_repair.py` still exercise only the older `search_and_link`-only flow.

## Topics to Explore

- [file] `reasons/repair.py` — Read the truncated tail (lines ~500–618) to confirm the exact statuses of `repair_smuggled_beliefs` and whether `research_beliefs = repair_beliefs` is intended as a public alias or deprecation shim
- [function] `reasons/repair.py:repair_beliefs` — Trace each branch's `dry_run` semantics and confirm metadata side effects are truly skipped in dry-run for all four patterns
- [function] `reasons/repair.py:_do_search_and_link` — Verify the candidate filter (`truth_value == "IN"`, not `belief_id`, not in existing `antecedents`, not derived) against how `api.search` returns results
- [file] `tests/test_repair.py` — The tests currently only cover `repair_smuggled_beliefs`; check whether triage-based `repair_beliefs` (all four patterns) has test coverage elsewhere
- [general] `metadata-taxonomy` — Enumerate all `set_metadata` keys used across repairs (`repair_action`, `repair_research`) and how downstream CLI/report commands read them

## Beliefs

- `search-and-link-preserves-outlist` — `_do_search_and_link` passes the original `outlist` through as the `unless` argument to `api.add_justification` so the blocking set is unchanged.
- `soften-writes-metadata-to-new-node` — The `soften` branch sets `repair_action=softened` on `sup["new_id"]`, not on the original `belief_id`.
- `research-sets-two-metadata-keys` — The `research` branch is the only pattern that writes both `repair_action=research` and `repair_research=<triage rationale>`.
- `abandon-embeds-rationale-in-retract-reason` — The `abandon` branch interpolates the triage rationale into the `retract_node` `reason` string in addition to `set_metadata`.
- `dry-run-still-reports-success-status` — Every pattern sets its success status (`linked`/`softened`/`abandoned`/`needs_research`) regardless of `dry_run`; only the `api.*` write calls are gated by `if not dry_run`.
- `repair-beliefs-statuses-differ-from-smuggled` — `repair_beliefs` uses `linked` for a successful search-and-link while the older `repair_smuggled_beliefs` (as exercised in `tests/test_repair.py`) uses `repaired`.
