---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: The cross-file contract shared by `.forge/report.json`, `.forge/proposed-entries.json`, and `costs/*-update.json`; worth pinning down as one schema rather than three ad hoc files.

## What these files actually are

Each file is a different *view* of the same `diary-update` pipeline run — but they record different dimensions of it, and they live under different roots and lifecycles.

**`.forge/report.json`** is the run's narrative summary: `pipeline` (`"diary-update"`), `started_at`/`updated_at`/`completed_at`, a `status`, a `steps[]` array where each step carries `name`, `status`, its own timestamps, a `detail` string, a `counts` object, and `duration_seconds`, plus a top-level `summary`. In the sample, `summary` is exactly the union of the two steps' `counts` blocks (`entries: 17`, `proposed: 260`, `accepted: 0`, `total_beliefs: 201`).

**`.forge/proposed-entries.json`** is *not* a run record at all — it's a content-addressed dedup ledger. It is a flat map of `entry path → 16-char hash` (8 bytes of hex). It contains exactly 17 keys, matching `summary.entries: 17`, which is the strongest observable contract between the two `.forge/` files: the report counts entries, this file *is* the entry set.

**`costs/2026-11-19T180331-update.json`** is a per-run cost/delta record. It carries a single `timestamp`, an `operation` (`"update"`), a `domain` (null here), a primary `model`, a top-level rollup (`calls`, `input_tokens`, `output_tokens`, `cost_dollars`), a `by_model` map with the same five fields, and belief deltas (`beliefs_added`/`retracted`/`unchanged`). The filename encodes the same timestamp (`180331`) plus the operation (`update`), so this file is **append-only**: one immutable file per run.

## The joints that make the "one schema" argument plausible

There is a real shared contract, and it has four axes:

1. **A timestamp identity.** `report.json` uses offset-aware ISO 8601 (`2026-11-22T18:00:02+00:00`). The cost log uses naive `2026-11-19T18:03:31` and encodes it in the filename with colons stripped. A single schema would need to pick a `datetime` convention; the codebase currently does not.

2. **A rollup idiom.** Both the report and the cost log distinguish a *detail array* from a *top-level aggregate*: `steps[].counts` → `summary`, and `by_model[model] → {…}` → the top-level `calls`/`tokens`/`cost`. This is the same "detail/rollup" pattern under two different names and could be folded into one reusable `RunRollup` shape.

3. **An operation vocabulary.** `pipeline: "diary-update"` in the report maps to `operation: "update"` in the cost file's `operation` field and `-update` in its filename. `proposed-entries.json` has no operation field — it's the implicit output of the `propose-beliefs` step named inside `report.json.steps[0]`. This vocabulary is only consistent if you assume the writer of each file derived it from the same pipeline config.

4. **A shared count field.** `summary.entries` (report) equals the number of keys in `proposed-entries.json`. `beliefs_added + retracted + unchanged` in the cost file is the delta-log counterpart to `report.json`'s `counts.accepted` and `counts.total_beliefs`.

## Where the "one schema" hypothesis breaks

Two places, both important:

- **Lifecycle differs.** `report.json` is a singleton that gets overwritten every run ("latest state"). `costs/*-update.json` is append-only (one immutable file per run, timestamp+operation in the filename). `proposed-entries.json` is a *mutable* ledger whose hash values change as entries change. A schema that collapses all three into one record type would silently drop the lifecycle distinction — which is exactly the distinction that makes the `.forge/` files idempotent next run.

- **Root directories differ.** `.forge/` vs `costs/`. That is not cosmetic — it likely reflects that cost files are user-auditable output while `.forge/` files are workspace state.

## What the observations do *not* show

The observations are **insufficient** to pin down the writer/reader code. Specifically:

- `find_report_json_usage` returned **zero matches**, and `find_proposed_entries_usage` returned **zero matches**. No code in the gathered snippets reads or writes either file.
- `reasons/llm.py` is only referenced via its docstring mention of "costs" at line 11 and the module comment at line 124 — the actual cost-log writer is not in the observations.
- `reasons/propose_update.py` and `reasons/report_belief.py` are *not* the writers of these three files. `propose_update.py` (lines 44–77, `PROPOSE_UPDATE_PROMPT`) defines a belief-update taxonomy (`action`/`failure_mode`/`basis`, with `FAILURE_MODES` and `BASES` on lines ~17–31) — an entirely different subsystem from a `diary-update` entry-proposal pipeline. `report_belief.py` produces structured markdown (`_build_structured_report`), not `report.json`. Anyone reading this code looking for the writers will be misled.
- `reasons/propose_update.py:parse_update_proposals` (lines ~144–200) uses `json.JSONDecoder.raw_decode` in a loop to tolerate prose around arrays, then coerces `action`, `failure_mode`, and `basis` into the allowed sets. This is the project's existing *pattern* for enforcing a schema on LLM output — it is the closest thing to a precedent for "pin down one schema" — but it governs belief proposals, not the `.forge/` artifacts.

So: the shared contract is **observable in the data shapes but not enforceable from the shown code**. To actually collapse these into one schema you'd need the file writers, the readers that consult `proposed-entries.json` on the next run to decide idempotency, and the cost-aggregation function that consumes `costs/*-update.json`.

## Topics to Explore

- [general] `diary-update-pipeline-entrypoint` — The `pipeline: "diary-update"` and steps `propose-beliefs`/`accept-beliefs` in `report.json` must be orchestrated somewhere; find that function to see where the three artifacts are written.
- [file] `reasons/llm.py` — Contains the only code hits for "costs"; likely owns the cost-log writer and the `by_model` rollup.
- [general] `.forge-directory-writer` — Search for the literal `.forge/` path in the repo to locate where `report.json` and `proposed-entries.json` are created and where `proposed-entries.json` is consulted for idempotency.
- [function] `reasons/propose_update.py:parse_update_proposals` — Demonstrates the existing LLM-output coercion pattern (raw_decode + set membership) that a unified schema would need to mirror for entry proposals.
- [general] `report-summary-vs-step-counts` — Verify the invariant `summary == ∪(steps[].counts)` holds for other runs by inspecting more `report.json` samples; if it does, that aggregation belongs in the schema.

## Beliefs

- `proposed-entries-json-is-content-addressed-dedup-ledger` — `.forge/proposed-entries.json` maps entry path → 16-char hash and contains no timestamp or operation field, so it must be re-hashed on every run to decide idempotency.
- `report-json-summary-aggregates-step-counts` — `report.json.summary` is the union of the `counts` objects across `steps[]` (verified: `entries`/`proposed` come from `propose-beliefs`, `accepted`/`total_beliefs` from `accept-beliefs`).
- `cost-update-json-is-append-only` — `costs/2026-11-19T180331-update.json` encodes its timestamp and `operation` in the filename, so each run produces a distinct immutable file rather than overwriting one.
- `cost-update-json-operation-maps-to-report-pipeline` — `operation: "update"` in `costs/*-update.json` corresponds to `pipeline: "diary-update"` in `.forge/report.json` for the same run.
- `report-json-summary-entries-equals-proposed-entries-key-count` — `summary.entries: 17` in `report.json` matches the 17 keys in `proposed-entries.json`.
- `report-json-carries-dual-completion-timestamps` — `report.json` records both `updated_at` and `completed_at`; they coincide in the sample but are semantically distinct (in-progress vs terminal).
