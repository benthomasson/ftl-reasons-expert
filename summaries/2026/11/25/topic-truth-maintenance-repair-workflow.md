---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: How review → triage → repair loops are invoked from the CLI and what state they persist in the DB.

# Review → Triage → Repair: CLI Invocation & Persisted DB State

## 1. How the loop is shaped

The codebase implements the loop in three separate layers, and only the last layer writes to the DB:

| Stage | Where | Persists? |
|---|---|---|
| Review | `reasons/review.py` — `review_beliefs()` | **No.** Returns a list of dicts only. |
| Triage | `reasons/repair.py` — `triage_belief()` (LLM chooses a pattern) | **No.** Pattern is returned in-memory. |
| Repair | `reasons/repair.py` — `repair_beliefs()` dispatches to 4 patterns | **Yes**, via `api.add_justification`, `api.supersede_with_text`, `api.retract_node`, and `api.set_metadata`. |

## 2. CLI invocation — what the observations actually show

**The observations cannot confirm a CLI entry point.** The evidence:

- `cli_review_triage_repair_commands` — searched `reasons/cli.py` for `review|triage|repair`, and returned **0 matches**.
- `read_cli_file` — only **lines 1–800 of 4015** were read. The visible `cmd_*` functions are `init`, `add`, `add-justification`, `remove-justification`, `retract`, `mark-duplicate`, `mark-superseded`, `defeat-justification`, `defeat-with-scope`, `migrate-defeaters`, `classify-defeaters`, `assert`, `what-if`, `status`, `show`, `tag`, `untag`, `tags`, `add-source`, `explain`, `convert-to-premise`, `summarize`, `supersede`, `set-metadata`, `get-metadata`, `challenge`, `defend`, `nogood`. **None of them are review/triage/repair.**
- `find_triage_symbol` — 0 definitions of a symbol named exactly `triage`; the triage step is actually named `triage_belief` in `reasons/repair.py` and was missed by the exact-name search.
- `find_repair_symbol` — **one** hit: `reasons/api.py:4684  def repair(`. So the public API surface exists, but the CLI wrapper that calls it is **not in the observed window**.

**Conclusion:** From the observations, the loop is invoked through the public API function `reasons/api.py:repair` (line 4684). The CLI command that wraps it is almost certainly in lines 801–4015 of `reasons/cli.py` (or is named something else, e.g. `audit`/`validate`/`upgrade`), but that portion of the file was not inspected, so the exact subcommand name and any argument surface are unverified.

## 3. What the loop actually does

### Step 1 — Review: `reasons/review.py:review_beliefs`

- Iterates sorted nodes that are `IN` and have at least one justification (`review.py:review_beliefs`).
- Batches at `REVIEW_BATCH_SIZE = 20`.
- For each batch, calls `invoke_model(prompt, ...)` and parses the LLM JSON with `parse_review_response`.
- Each result is a dict with keys: `id`, `valid`, `sufficient`, `necessary`, `unnecessary_antecedents`, `comment`, `scope_findings`, `missing_property`, `defeat_reason_type` (see `parse_review_response`, `review.py`).
- Ancillary helpers: `format_belief_for_review` (formats each belief + antecedent texts), `DEFEAT_REASON_TYPES`, `classify_defeat_reason` (the LLM classifier used by `classify-defeaters`).

**No DB calls appear in `review.py`.** Review verdicts live only in the returned list.

### Step 2 — Triage: `reasons/repair.py:triage_belief`

Inside `repair_beliefs`, each invalid result (`r["valid"] == False`) is triaged:

- `belief_context = format_belief_for_review(belief_id, nodes)`
- `depth = _compute_depth(belief_id, nodes)` (0 for premises, `max(antecedent depth) + 1` otherwise)
- `flagged_ancestors` = count of direct antecedents that were also in `invalid_ids`
- `triage_belief(...)` uses `TRIAGE_PROMPT` and returns `{"pattern": ..., "rationale": ...}` where `pattern ∈ VALID_PATTERNS = {search_and_link, soften, abandon, research}`.

Again, **nothing is persisted at triage time** — `result["pattern"]` is populated in the local return dict only.

### Step 3 — Repair: `repair_beliefs` dispatch (`repair.py`, ~line 380)

| Pattern | What it does | DB write(s) |
|---|---|---|
| `search_and_link` (`_do_search_and_link`) | Extract "smuggled claim" (`EXTRACT_PROMPT`), search via `api.search`, match with `MATCH_PROMPT`, filter to IN premises | `api.add_justification(belief_id, sl=new_ants, unless=..., label=f"repair: linked {claim[:50]}")` + `api.set_metadata(belief_id, "repair_action", "search_and_link")` |
| `soften` | `SOFTEN_PROMPT` rewrites text to match antecedents | `api.supersede_with_text(belief_id, softened)` → creates `new_id`, then `api.set_metadata(new_id, "repair_action", "softened")` |
| `abandon` | No LLM rewrite — just retract | `api.retract_node(belief_id, reason=f"repair: abandoned — {rationale}")` + `api.set_metadata(belief_id, "repair_action", "abandoned")` |
| `research` | Flags for human follow-up; **keeps cardinality** | `api.set_metadata(belief_id, "repair_research", rationale)` + `api.set_metadata(belief_id, "repair_action", "research")` |

The `dry_run` flag (default false) is set from the caller (`dry_run` kwarg); when true, the LLM steps still run but all persistence calls are skipped (branch guards `if not dry_run:`).

Aliases / related entry points:
- `research_beliefs = repair_beliefs` (`repair.py`, near line 505) — same orchestrator.
- `repair_smuggled_beliefs` (`repair.py`, after `repair_beliefs`) — earlier/alternative orchestrator focused only on search-and-link.
- `reasons/api.py:4684  def repair(` — the public API function the CLI most likely calls.

## 4. What state is persisted in the DB

Because `review_beliefs` and `triage_belief` return plain dicts, **review verdicts and triage rationales are ephemeral unless the caller writes them**. The only durable repair-related state observed is the `set_metadata` calls above. Concretely:

| Key | Set by | Values seen |
|---|---|---|
| `repair_action` | every non-abandoned, non-dry-run path | `search_and_link`, `softened`, `abandoned`, `research` |
| `repair_research` | `research` pattern | free-form rationale from triage |

Plus the standard TMS mutations that are the actual effects:
- New justification row (search-and-link)
- New node + `superseded_by` (soften)
- `retracted_at` / `truth_value=OUT` (abandon)
- (research does **not** change truth value — only metadata)

**Caveat on the DB-grep observations:** `db_review_repair_state_grep` and `pg_review_repair_state_grep` both returned `total_count: 0` for `review|triage|repair|status|state|CREATE TABLE|INSERT INTO|UPDATE`. That is inconsistent with the `set_metadata` calls seen in `repair.py`, which means one of:
1. The grep target file is not `reasons/db.py`/`reasons/pg.py` (or those files use a generic metadata key/value table whose SQL doesn't contain the word "repair").
2. The regex was applied to a truncated file or executed with escaping issues.
3. `set_metadata` is implemented via a JSON blob column, so `review`/`repair` strings appear only as values, not as schema/column names.

**Therefore the precise schema is unverified.** Based on the API surface, `set_metadata`/`show_node` (which prints arbitrary `metadata` keys, see `cmd_show` in `cli.py`) implies a generic `metadata` map attached to a node (likely a JSONB-style column or a `node_metadata` table), not dedicated review/triage/repair tables.

## 5. What the observations are missing

- **Lines 801–4015 of `reasons/cli.py`** — the CLI argument definitions and `cmd_*` handlers that actually invoke `api.repair`. Without this, the exact subcommand name (`reasons review`? `reasons repair`? `reasons audit`?) is unknown.
- **`reasons/api.py:repair` body** — does it call `review_beliefs` then `repair_beliefs`, or does it only do one side? Does it pass `dry_run` through?
- **`reasons/db.py` / `reasons/pg.py`** schema — to confirm whether `set_metadata` writes to a JSONB column, a `node_metadata` table, or a `node` column, and whether `review` verdicts are ever persisted.
- **`reasons/api.py:set_metadata`** — the exact write semantics.
- **`reasons/api.py:supersede_with_text`** — confirms that soften creates a new node with `superseded_by` metadata.

## Topics to Explore

- [file] `reasons/cli.py` — Lines 801–4015 contain the actual CLI wiring for review/repair; the observed window ends at line 800.
- [function] `reasons/api.py:repair` — The public API function (line 4684) that the CLI presumably invokes; its body determines whether review and repair are chained in one call.
- [function] `reasons/api.py:set_metadata` — Defines what "repair_action" / "repair_research" actually persist to, which resolves the contradiction in `db_review_repair_state_grep`.
- [general] `review-result-persistence` — Whether the review verdicts (`valid`, `scope_findings`, `defeat_reason_type`) are ever written back to the DB or are purely consumed by the in-memory `repair_beliefs`.
- [function] `reasons/repair.py:repair_beliefs` — The orchestrator that ties triage and all four repair patterns together; verify how `dry_run` and the `search_fn` override propagate.

## Beliefs

- `cli-review-repair-commands-not-in-first-800-lines` — `reasons/cli.py` lines 1–800 contain no `cmd_*` function matching `review|triage|repair`; the wrapper for `api.repair` lies in lines 801–4015 (unread).
- `review-beliefs-does-not-persist` — `reasons/review.py:review_beliefs` returns a list of review dicts and contains no `api`/DB calls.
- `repair-implements-triage-then-dispatch` — `repair_beliefs` calls `triage_belief` (LLM) before dispatching to one of the four `VALID_PATTERNS`.
- `search-and-link-produces-add-justification` — The `search_and_link` pattern persists via `api.add_justification` and then writes `repair_action="search_and_link"` metadata.
- `soften-produces-supersede-with-text` — The `soften` pattern creates a new node via `api.supersede_with_text` and writes `repair_action="softened"` on the new node.
- `research-persists-metadata-only` — The `research` pattern writes `repair_research` and `repair_action="research"` metadata but does not change truth value or retract.
- `repair-state-is-metadata-not-tables` — The only repair-related writes observed are `set_metadata` calls; no dedicated review/triage/repair tables appear in `repair.py`.
