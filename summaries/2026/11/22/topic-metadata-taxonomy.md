---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Enumerate all `set_metadata` keys used across repairs (`repair_action`, `repair_research`) and how downstream CLI/report commands read them

## Repair metadata writes

Only two repair-specific `set_metadata` keys appear in the observations: `repair_action` and `repair_research`.

| Key | Value(s) | Written where | Target node | Conditions / notes |
|---|---|---|---|---|
| `repair_action` | `rewritten` | `reasons/api.py:4431` | `sup["new_id"]` | Repair/supersede path in `api.py`; successor node gets the marker. |
| `repair_action` | `retracted` | `reasons/api.py:4440` | `nid` | Repair/retract path in `api.py`. |
| `repair_action` | `search_and_link` | `reasons/repair.py:438` | `belief_id` | Only after search-and-link succeeds with status `"linked"` and `not dry_run`. |
| `repair_action` | `softened` | `reasons/repair.py:458` | `sup["new_id"]` | After `supersede_with_text(...)`; marker is on the successor, not the original. |
| `repair_action` | `abandoned` | `reasons/repair.py:470` | `belief_id` | After `retract_node(...)`, inside `if not dry_run`. |
| `repair_action` | `research` | `reasons/repair.py:482` | `belief_id` | Set after `repair_research`; inside `if not dry_run`. |
| `repair_research` | Free-form triage rationale string | `reasons/repair.py:477-480` | `belief_id` | Stores `triage.get("rationale", "")`; set immediately before `repair_action=research`. |

So the effective `repair_action` value set is:

```text
rewritten
retracted
search_and_link
softened
abandoned
research
```

`repair_research` is not a status flag; it stores the LLM triage rationale that explains why the belief was placed into the `research` bucket.

Tests confirm these write contracts:

- `tests/test_repair_premises.py:332` asserts successor metadata has `repair_action == "rewritten"`.
- `tests/test_repair_premises.py:346` asserts `repair_action == "retracted"`.
- `tests/test_repair_premises.py:360` asserts dry-run leaves `repair_action` as `None`.
- `tests/test_research.py:236,267,292,319` asserts/mocks writes for `search_and_link`, `softened`, `abandoned`, and `research`.
- `tests/test_research.py:315` asserts `repair_research == "needs code review"`.

The writes go through `api.set_metadata`:

- `reasons/api.py:1218` defines `set_metadata(...)`.
- `reasons/api.py:1227` dispatches to PostgreSQL via `_pg_dispatch(...)`.
- `reasons/pg.py:1512` defines the PostgreSQL `set_metadata(self, node_id, key, value)` implementation.
- The generic CLI writer is `reasons/cli.py:731` `cmd_set_metadata`, registered at `reasons/cli.py:3984`; it calls `api.set_metadata(...)` but does not read repair keys.

`set_metadata` also updates `updated_at`; `tests/test_api.py:852-855` verifies that behavior. That means repair metadata writes indirectly affect timestamp display in commands that show `updated_at`.

## Downstream CLI/report reads

Based on the provided observations, **no downstream CLI or report command reads `repair_action` or `repair_research`**. The only references to those keys outside the write sites are tests. The `repair_metadata_key_references` result contains no matches in `reasons/cli.py`, `reasons/report_belief.py`, `reasons/report_gated.py`, or `reasons/export_markdown.py`.

Specific observations:

- `reasons/cli.py:cmd_show` prints selected metadata keys: `source_type`, `accepted_pr`, `pinned_sha`, `pinned_lines`, `retract_reason`, and `example`. It does **not** print `repair_action` or `repair_research`.
- `reasons/cli.py:cmd_status` uses API summary fields such as counts and node listing fields. It does not read repair metadata in the observed snippet.
- `reasons/cli.py:cmd_mark_duplicate` and `cmd_mark_superseded` print `duplicate_of` and `superseded_by`, not repair metadata.
- `reasons/cli.py:cmd_retract` and `cmd_migrate_defeaters` work with `retract_reason`, not `repair_action`.
- `reasons/export_markdown.py` reads metadata fields including `stale_reason`, `retract_reason`, `beliefs_type`, `source_type`, `duplicate_of`, `superseded_by`, `accepted_pr`, `defeats_node`, `defeater_type`, and `defeat_reason_type`. It does **not** export `repair_action` or `repair_research`.
- `reasons/report_belief.py` builds reports from `node_detail`, `explain_steps`, `premises_data`, and `source_chunks`; it does not read metadata repair keys.
- `reasons/report_gated.py` consumes `retracted_premises` and reads `retract_reason` for fixed defects. It does not read `repair_action` or `repair_research`.

Important implication: an abandoned repair may be indirectly visible through `retract_reason`, because `repair.py` calls `api.retract_node(..., reason="repair: abandoned — ...")`, and `cmd_show`, `export_markdown`, and `report_gated` do read `retract_reason`. But the explicit marker `repair_action=abandoned` is not surfaced by those commands. Similarly, `repair_research` rationale is stored but not displayed by observed CLI/report paths.

To fully prove there are no other consumers, a full grep for `repair_action` and `repair_research` across the entire repository is needed, including dynamic metadata access patterns. The provided observations are sufficient to say the observed CLI/report modules do not read them.

## Topics to Explore

- [file] `reasons/api.py` — Inspect the full repair functions around lines 4431 and 4440 to confirm the `rewritten` and `retracted` contexts and whether any other repair metadata is written nearby.
- [function] `reasons/cli.py:cmd_show` — Verify exactly which metadata fields the CLI exposes; currently `repair_action` and `repair_research` are not printed.
- [file] `reasons/export_markdown.py` — Check whether repair metadata should be included in markdown exports; observed export ignores it.
- [general] `repair-metadata-consumers` — Run a full repository grep for `repair_action` and `repair_research` to confirm whether downstream commands or reports consume them dynamically.
- [function] `reasons/repair.py:repair_beliefs` — Trace dry-run and success conditions for each repair action to understand when metadata is written versus skipped.

## Beliefs

- `repair-action-is-write-only-in-observed-code` — No observed CLI or report module reads `repair_action` or `repair_research`; only repair code and tests reference them.
- `repair-action-has-six-observed-values` — `repair_action` is written as `rewritten`, `retracted`, `search_and_link`, `softened`, `abandoned`, or `research`.
- `repair-research-stores-triage-rationale` — The `research` branch writes the triage rationale string to `repair_research` before setting `repair_action=research`.
- `repair-metadata-writes-respect-dry-run` — Repair metadata writes occur inside `not dry_run` guards; dry-run tests assert `repair_action` is not set.
- `set-metadata-is-backend-dispatched` — `api.set_metadata` delegates to PostgreSQL via `_pg_dispatch` and otherwise uses the local/SQLite path, while updating `updated_at`.
