---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Decide whether `UNKNOWN` should be rejected, parsed as a claim, or mapped to `OUT`.

## Decision

`UNKNOWN` should **not** be rejected globally, **not** parsed as a claim status, and **not** mapped to `OUT`.

Instead, treat `UNKNOWN` as a **presentation/default sentinel for missing truth-value data**. It should remain a report-layer fallback when `truth_value` is absent, while the RMS/network domain should continue using explicit statuses such as `IN`, `OUT`, and import-only `STALE`.

If `UNKNOWN` appears where a claim status is required, then reject/skip it as an unsupported claim status rather than accepting it as a valid claim. But in the current code, `UNKNOWN` is not really a claim status at all.

## Evidence

- `find_unknown_symbol` found **zero definitions** of `UNKNOWN`. This means `UNKNOWN` is not a shared constant, enum member, or formal status symbol in this codebase. It appears only as string literals.

- In `reasons/report_belief.py`, `UNKNOWN` is used as a `dict.get` fallback:
  - line 38: `status = node_detail.get("truth_value", "UNKNOWN")`
  - line 85: `tv = pd.get("truth_value", "UNKNOWN")`
  - lines 123, 149, 194: same pattern in other report paths.

  This strongly indicates `UNKNOWN` means “the field was missing,” not “the RMS assigned this node the status UNKNOWN.”

- In `_build_structured_report` and `_format_data_for_prompt` in `reasons/report_belief.py`, truth values are rendered roughly as:
  - `+` if `truth_value == "IN"`
  - `-` otherwise.

  So `UNKNOWN` currently falls into the non-`IN` branch. That is a display behavior, not a semantic mapping to `OUT`. However, it does mean the report marker can visually conflate `UNKNOWN` with `OUT`. If that distinction matters, change the renderer to show `?` or `UNKNOWN` explicitly rather than mapping it to the same marker as `OUT`.

- In `reasons/api.py`, lines 3764 and 3836 append premise placeholder data like:
  ```python
  {"id": pid, "text": "", "truth_value": "UNKNOWN", "source": ""}
  ```
  This is a report/payload placeholder for missing premise details. It does not appear to be claim parsing or network truth maintenance assignment.

- In `reasons/import_beliefs.py`, `parse_beliefs` recognizes claim headers only with:
  ```python
  ^### (\S+) \[(IN|OUT|STALE)\]\s*(.*)
  ```
  So an `[UNKNOWN]` claim header would not be parsed as a claim. The importer maps:
  - `IN` with no dependencies → premises,
  - `IN` with dependencies → SL-justified nodes,
  - `STALE` and `OUT` → nodes retracted after import.

  There is no `UNKNOWN` branch. Therefore parsing `UNKNOWN` as a claim would be a schema change, and mapping it to `OUT` would conflate “unknown/missing” with “retracted/not believed.”

- In `reasons/derive.py`, `_build_beliefs_section` filters only exact `IN` nodes:
  ```python
  v.get("truth_value") == "IN"
  ```
  and `_build_derived_section` uses:
  ```python
  status = nodes[k].get("truth_value", "?")
  ```
  This suggests missing truth-value status is displayed as `?` in derive, not as `UNKNOWN`, and that `UNKNOWN` is not part of the derive/network truth-value model.

- `grep_unknown_tests` found no tests. So current behavior for missing/unknown truth values is untested. If you change the rendering or validation behavior, add tests.

- The `grep_out_status_reasons` summary reports zero matches for `\bOUT\b`, but the provided `reasons/import_beliefs.py` content clearly contains `OUT` in the parser and import mapping. Treat that grep summary as incomplete or mis-scoped for this decision. The code excerpt is stronger evidence that `OUT` is a real import/claim status.

## Recommendation

Use `UNKNOWN` only as a **display fallback** for missing truth values.

Do not:
- add `UNKNOWN` to the claim parser’s accepted statuses,
- map `UNKNOWN` to `OUT`,
- persist `UNKNOWN` into the RMS as a real truth value without an explicit schema change.

If a report needs to distinguish `OUT` from missing data, update the report rendering so:
- `IN` → `+`
- `OUT` → `-`
- missing / `UNKNOWN` → `?` or `UNKNOWN`

If an input claim file contains `[UNKNOWN]`, reject or skip it with a clear error rather than silently treating it as a valid claim.

## Topics to Explore

- [file] `reasons/api.py` — Inspect the full context around lines 3764 and 3836 to confirm whether `UNKNOWN` placeholders ever enter the network or only the report payload.
- [function] `reasons/import_beliefs.py:parse_beliefs` — Confirm how unsupported claim statuses such as `[UNKNOWN]` are handled and whether they are silently skipped.
- [file] `reasons/report_belief.py` — Review marker rendering for non-`IN` truth values and decide whether `UNKNOWN` should be visually distinct from `OUT`.
- [general] `truth-value-domain` — Locate the canonical definitions/validators for `IN`, `OUT`, `STALE`, and missing truth values across the network and CLI.
- [general] `unknown-status-tests` — Add tests for missing `truth_value` fields, unsupported claim statuses, and report rendering of missing vs. `OUT` nodes.

## Beliefs

- `unknown-is-a-report-default` — `UNKNOWN` is used in `reasons/report_belief.py` as a `dict.get("truth_value", ...)` fallback, not as an assigned network status.
- `import-parser-rejects-unknown-claim-status` — `parse_beliefs` in `reasons/import_beliefs.py` only recognizes claim headers with status `IN`, `OUT`, or `STALE`.
- `out-means-retracted-on-import` — `import_into_network` retracts imported claims whose parsed status is `STALE` or `OUT`.
- `derive-filters-in-only` — `_build_beliefs_section` in `reasons/derive.py` includes only nodes whose `truth_value` is exactly `"IN"`.
- `report-marker-conflates-unknown-and-out` — In `_build_structured_report`, `marker = "+" if tv == "IN" else "-"` renders any non-`IN` value, including `UNKNOWN`, with the same minus marker as `OUT`.
