---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Which `defeat_reason_type` values are meaningful in this system and how downstream reports group on them.

## What `defeat_reason_type` is

`defeat_reason_type` is an optional metadata classification on defeater nodes, described in `reasons/api.py` as a **“Logical failure mode classification”** (`reasons/api.py:1343`, `reasons/api.py:1440`).

The meaningful non-empty values come from `DEFEAT_REASON_TYPES` in `reasons/review.py`, which is imported by the classification tests:

- `tests/test_classify_defeaters.py` imports `DEFEAT_REASON_TYPES` and asserts there are exactly **7** values.
- That same test asserts each value contains a hyphen.
- Observed examples of valid/meaningful `defeat_reason_type` values are:
  - `unsupported-conjunct`
  - `over-generalizes`
  - `scope-mismatch`

The **full seven-value set is not shown in the observations**. To get the canonical list, inspect `reasons/review.py` directly.

## How it gets assigned

The batch classification flow is:

- `reasons/api.py:1652` — `classify_defeat_reason_types(...)`
- `reasons/api.py:1685` — skips nodes that already have a truthy `defeat_reason_type`
- `reasons/api.py:1702` — calls `classify_defeat_reason(...)`
- `reasons/api.py:1712` — appends `{"id": nid, "defeat_reason_type": reason_type}`
- `reasons/api.py:1715` — writes metadata via `set_metadata`

`classify_defeat_reason` returns `""` when the LLM output is unrecognized or errors, per `tests/test_classify_defeaters.py`. So:

- **Non-empty members of `DEFEAT_REASON_TYPES` are meaningful classifications.**
- **Empty string `""` means “not classified / unknown”.**
- The batch classifier only skips already-classified nodes when `meta.get("defeat_reason_type")` is truthy, so `""` does **not** count as already classified.

Also distinguish this from `defeater_type`:

- `defeater_type` values appear in `tests/test_defeat_justification.py`, e.g. `invalid-inference`, `over-generalizes`, `duplicate-of`, `superseded-by`.
- `defeat_reason_type` is a separate classification assigned later by `classify_defeat_reason_types`.
- `classify_defeat_reason_types` can filter by `defeater_type`, as shown in `tests/test_classify_defeaters.py:test_filters_by_defeater_type`.

## How downstream reports group on it

Based on the provided observations, **the reports shown do not group by `defeat_reason_type`**.

- `reasons/report_gated.py` never reads `defeat_reason_type`.
  - It groups data by **retracted premises** and **active blockers**.
  - Fixed defects are sorted by `dependent_count`.
  - Active blockers are keyed by blocker ID and sorted by number of gated beliefs.
  - It uses `retract_reason`, not `defeat_reason_type`, in the fixed-defects table.
- `reasons/report_belief.py` never reads `defeat_reason_type`.
  - It traces a belief through its justification chain and root premises; it does not use or group by defeat-reason metadata.
- The grep for `group.*defeat|defeat.*group` returned **0 matches**.
- The CLI classification path prints classifications one-by-one:
  - `reasons/cli.py:343` — `print(f"  {c['id']}: {c['defeat_reason_type']}")`
  - This is a flat listing, not a grouped report.

Two files do read the field:

- `reasons/build_wiki.py:95` — `r_type = d_meta.get("defeat_reason_type", "")`
- `reasons/export_markdown.py:88` — `rtype = o_node.metadata.get("defeat_reason_type", "")`

However, the observations only show those reads, not the full rendering or grouping logic. So the observations are **insufficient to claim** that `build_wiki.py` or `export_markdown.py` group by `defeat_reason_type`; they clearly consume it, but the downstream behavior is not visible here.

## Topics to Explore

- [file] `reasons/review.py` — Contains `DEFEAT_REASON_TYPES`; this is the canonical source for the exact seven meaningful `defeat_reason_type` values.
- [function] `reasons/api.py:classify_defeat_reason_types` — Implements classification, skip/error behavior, filtering, dry-run, and metadata writes.
- [file] `reasons/build_wiki.py` — Reads `defeat_reason_type` at line 95; inspect how `r_type` is rendered or grouped.
- [file] `reasons/export_markdown.py` — Reads `defeat_reason_type` at line 88; inspect how `rtype` affects Markdown export.
- [general] `defeater-type-vs-defeat-reason-type` — Clarify the separate taxonomies: `defeater_type` describes the defeater creation reason; `defeat_reason_type` classifies the logical failure mode.

## Beliefs

- `defeat-reason-types-defined-in-review-py` — The canonical valid `defeat_reason_type` values are defined by `DEFEAT_REASON_TYPES` in `reasons/review.py`.
- `defeat-reason-types-has-seven-hyphenated-values` — Tests assert `DEFEAT_REASON_TYPES` contains exactly seven values and each contains a hyphen.
- `empty-defeat-reason-type-means-unclassified` — `classify_defeat_reason` returns `""` on unrecognized/error output, and batch classification only skips truthy existing metadata.
- `classify-defeat-reason-types-writes-metadata-when-not-dry-run` — Tests show dry-run classification does not write metadata, while apply mode writes `defeat_reason_type` via `set_metadata`.
- `provided-reports-do-not-group-by-defeat-reason-type` — `report_gated.py` and `report_belief.py` never reference `defeat_reason_type`, and no group-by-defeater grep matches were found.
