---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: How scope/missing-property findings become auditable defeat nodes in the graph

# How scope/missing-property findings become auditable defeat nodes

The mechanism is best read as a three-layer construction: **premise scope beliefs** → **derived defeater node** → **edge into the target justification's `outlist`**. Each layer leaves metadata that lets you audit both forward (who was defeated, when, and why) and backward (which scope findings drove this defeater). The test file `tests/test_defeat_with_scope.py` is the executable specification for the top-level helper; the primitive `defeat_justification` lives in `reasons/api.py`.

> Caveat: the observations give the *test* for `defeat_with_scope` and grep excerpts from `defeat_justification`, but not the full function body of `defeat_with_scope` or `reasons/review.py`. Line numbers below for `defeat_with_scope` come from the test file (behavioral), not the implementation.

## 1. Entry point — `api.defeat_with_scope`

A scope/missing-property finding is a dict of the shape `{antecedent, establishes, does_not_establish}`. A caller collects these (e.g. "premise-a establishes safety, does **not** establish traceability") plus a single `missing_property` ("traceability") and hands the set to `api.defeat_with_scope(node_id, justification_index, scope_findings=..., missing_property=..., db_path=...)`.

Error surface (from `test_defeat_with_scope_errors`):
- `KeyError("not found")` — unknown `node_id`
- `ValueError("no justifications")` — target has none
- `IndexError("out of range")` — `justification_index` too large
- `ValueError("scope_findings must not be empty")` — no findings

## 2. Artifact 1 — scope beliefs are first-class premise nodes

For each finding, a **new premise node** is materialized (`test_scope_beliefs_are_premises`):

- `truth_value == "IN"`
- `justifications == []` (i.e. a **premise**, not a derivation)
- `metadata["scope_of"] == "<antecedent>"` — back-pointer to the original antecedent
- `metadata["for_defeater"] == "<defeater_id>"` — **forward pointer to the defeater that consumes it**
- text embeds both sides of the finding verbatim: `"... establishes X"` and `"... does not establish Y"` (so `establishes`/`does_not_establish` survive even though they aren't asserted as discrete metadata keys)

IDs are unique per defeat call: `test_scope_ids_are_unique` shows two defeats on the same node produce two disjoint sets of scope IDs — scope beliefs are **anchored to a specific defeater**, not deduplicated globally.

## 3. Artifact 2 — the defeater is a **derived** SL node

`test_defeater_is_derived` shows the defeater is not a premise; it is a derived belief whose single justification is of type `"SL"` and whose `antecedents` are exactly the scope-belief IDs created in step 2. In other words, the defeater "follows from" the scope findings under the same truth-maintenance rules as any other belief.

Default ID: `{defeater_type}-{node_id}-j{justification_index}` (see `reasons/api.py:1371-1372`), overridable via `defeater_id=`.

Defeater metadata (from `test_defeater_is_derived` and `reasons/api.py:1374-1384`):

| key | meaning |
|---|---|
| `defeater_type` | e.g. `invalid-inference`, `over-generalizes` |
| `defeats_node` | which belief this defeater attacks |
| `defeats_justification` | which justification index |
| `defeat_reason_type` | optional logical failure-mode classification (see §5) |

Defeater text is `"{reason} (defeats {node_id} justification {justification_index})"` (`reasons/api.py:1374`).

## 4. Wiring — the defeater becomes the target's outlist entry

The primitive `defeat_justification` (`reasons/api.py:1322`) makes the graph edge explicit (`reasons/api.py:1386-1394`):

1. Create the defeater node (`net.add_node(id=defeater_id, ...)` at 1386).
2. Append `defeater_id` to the target justification's `outlist` (line 1392-1393) — this is what flips the target `IN`→`OUT` under the network's standard truth propagation.
3. Register a back-edge: `defeater.dependents.add(node_id)` (line 1394).

Because the edge is stored *on the justification*, not on the defeater, the target retains both a record of the defeat and the ability to recover when the defeater itself is retracted. `test_challenge_scope_restores_target` verifies this: retracting any scope belief makes the defeater `OUT`, which drops the target back to `IN`.

Cascading is symmetric with any other belief change — `test_cascade_through_scope_defeat` shows that defeating `d1` also flips `d2` (which depends on `d1`) to `OUT`.

## 5. Audit trail — the classification metadata layers

Two paths write `defeat_reason_type`, and both are deliberately conservative:

**Path A — synchronous, at defeat time.** Optional kwarg on `defeat_with_scope` / `defeat_justification`. Per `reasons/api.py:1383-1384`, the key is only written when the value is non-empty:
```python
if defeat_reason_type:
    meta["defeat_reason_type"] = defeat_reason_type
```
Confirmed by `test_defeat_reason_type_omitted` (key absent when empty) vs `test_defeat_reason_type_stored` (key present when set).

**Path B — post-hoc, LLM-assisted.** `reasons.review.classify_defeat_reason` (tested in `tests/test_classify_defeaters.py`) sends defeater text + defeated text to the model and only accepts a value from `DEFEAT_REASON_TYPES` (7 hyphenated types — `scope-mismatch`, `unsupported-conjunct`, `over-generalizes`, etc.). `api.classify_defeat_reason_types` is the batch driver:

- `dry_run=True` — returns proposed classifications in `result["classified"]` but **writes nothing** (verified: metadata still absent).
- `dry_run=False` — patches metadata onto the defeater node.
- **Idempotent** — nodes that already carry `defeat_reason_type` land in `result["skipped"]` with reason `"already classified"`.
- `defeater_type_filter=` narrows which defeater types are eligible.
- Non-defeater nodes are skipped (no `defeater_type` metadata).
- LLM failure → `result["errors"]`, never a crash.

CLI surface: `cmd_classify_defeaters` at `reasons/cli.py:322`, dispatched via `"classify-defeaters"` at `reasons/cli.py:3935`.

## 6. Portability — defeat nodes survive export/import

`test_defeat_with_scope_in_export` round-trips the whole artifact set through `api.export_network` / `api.import_json`:

- Both scope belief IDs and the defeater ID are present in `export["nodes"]`.
- The defeater's single SL justification keeps its `antecedents == [scope_id]`.
- After re-import into a fresh DB, `d1["truth_value"] == "OUT"` — the defeat is reconstructed purely from serialized graph state (no side-tables).

This is the core "auditable" property: everything needed to justify and reproduce the defeat is in the exported graph itself.

## 7. Summary of the pipeline

```
scope_findings + missing_property
        │
        ▼
[scope beliefs: IN premises, metadata {scope_of, for_defeater}]
        │  (antecedents)
        ▼
[defeater node: SL justification, metadata {defeater_type, defeats_node, defeats_justification, [defeat_reason_type]}]
        │  (outlist edge)
        ▼
[target justification.outlist ← defeater_id]
        │
        ▼
    target OUT  →  dependent beliefs OUT
        │
        ▼   (optional, later)
LLM classifier fills defeat_reason_type in metadata
```

## What the observations do *not* contain

- The implementation body of `defeat_with_scope` (only the behavioral spec via `tests/test_defeat_with_scope.py`).
- `reasons/review.py` source for `classify_defeat_reason` / `classify_defeat_reason_types` / `DEFEAT_REASON_TYPES` (only the CLI + test surface).
- The truth-propagation code that translates an `outlist` entry into an `OUT` value.
- The schema/store layer that persists node metadata across export/import.

## Topics to Explore

- [file] `reasons/api.py` — Full implementation of `defeat_with_scope` and `defeat_justification` (around line 1322); confirms ID generation, scope-belief text format, and metadata wiring
- [file] `reasons/review.py` — `classify_defeat_reason`, `classify_defeat_reason_types`, and the `DEFEAT_REASON_TYPES` taxonomy (7 hyphenated types)
- [function] `reasons/cli.py:cmd_classify_defeaters` — CLI glue at line 322; dispatch at `reasons/cli.py:3935`; argument parsing and output formatting for dry-run vs apply
- [general] `truth-propagation` — How an `outlist` entry on a justification propagates `IN`→`OUT` through the graph (needed to understand the cascade tests)
- [general] `export-import-roundtrip` — `api.export_network` / `api.import_json` schema, to see how metadata like `scope_of`, `for_defeater`, `defeats_node` survives serialization

## Beliefs

- `scope-beliefs-are-premises` — Scope beliefs created by `defeat_with_scope` have `truth_value == "IN"`, an empty `justifications` list, and metadata keys `scope_of` and `for_defeater`, per `test_scope_beliefs_are_premises`
- `defeater-is-a-derived-SL-node` — The defeater node always has exactly one `SL` justification whose `antecedents` are the scope-belief IDs (`test_defeater_is_derived`)
- `defeat-justification-requires-a-writable-outlist` — `reasons.api.defeat_justification` raises `ValueError` when the target has no justifications (line 1361) and appends the defeater ID onto `justification.outlist` (line 1392-1393)
- `defeat-reason-type-metadata-is-conditional` — `defeat_reason_type` is written to metadata only when non-empty (`reasons/api.py:1383-1384`), verified by `test_defeat_reason_type_omitted` vs `test_defeat_reason_type_stored`
- `classify-defeat-reason-types-skips-already-classified` — `api.classify_defeat_reason_types` returns already-classified defeaters under `skipped` with reason `"already classified"` rather than overwriting (`test_skips_already_classified`)
- `classify-defeat-reason-types-dry-run-writes-nothing` — With `dry_run=True`, classifications appear in the result but the defeater's `metadata["defeat_reason_type"]` remains absent (`test_dry_run_classifies_without_writing`)
