---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: How exported markdown and nogood sections are parsed back into a `Network`, including `duplicate_of` and stale metadata handling.

## How exported markdown and nogood sections are parsed back into a `Network`

The observations contain the **exporter**, API import call sites, and round-trip tests, but **not the actual implementation of `reasons/import_beliefs.py`** (`parse_beliefs`, `import_into_network`, `_normalize_nogoods_markdown`, etc.). So the parsing algorithm itself cannot be fully reconstructed from this data. What can be confirmed is the import pipeline, the exported format the parser must consume, and which round-trip behaviors are tested or known to be broken.

### 1. Import pipeline

The API import path reads `beliefs.md`, optionally reads a separate nogoods file, and falls back to a sibling `nogoods.md` if present:

- `reasons/api.py` around lines 1890–1921 reads `beliefs_text`, optional `nogoods_text`, and then calls `import_into_network(net, beliefs_text, nogoods_text)`.
- The same API documents the import result shape as:
  `{"claims_imported": int, "claims_skipped": int, "claims_retracted": int, "nogoods_imported": int}` at `reasons/api.py:1896`.
- The JSON import path uses `_normalize_nogoods_json` and `_normalize_nogoods_markdown` at `reasons/api.py:1950`.

Tests import directly from `reasons.import_beliefs`:

```python
from reasons.import_beliefs import import_into_network, parse_beliefs
```

`parse_beliefs(text)` parses claim sections into dictionaries. For example, `tests/test_roundtrip.py` asserts that `parse_beliefs` extracts:

- `source_url` from `- Source URL: ...`
- `accepted_pr` from `- Accepted PR: ...`

`import_into_network(net, md)` then builds/updates the `Network`. Tests confirm that exported `- Depends on:` and `- Unless:` bullets are reconstructed into justifications:

- `test_unless_round_trip` verifies `- Unless: b` becomes `justifications[0].outlist == ["b"]`.
- `test_derived_node` and `test_multiple_antecedents` show exporter format `- Depends on: a, b`.

### 2. Claim export format that the parser must consume

`reasons/export_markdown.py` writes each node as:

```markdown
### <node-id> [<STATUS>] <BELIEFS_TYPE>
<node text>
- Source: ...
- Source URL: ...
- Source type: ...
- Source hash: ...
- Date: ...
- Depends on: ...
- Unless: ...
- Defeated by: ...
- Stale reason: ...
- Duplicate of: ...
- Superseded by: ...
- Accepted PR: ...
```

Key details:

- Node ordering is IN first, then OUT, alphabetically inside each group.
- Status is normally the node truth value (`IN` / `OUT`).
- If the node is `OUT` and has `metadata["stale_reason"]` or `metadata["retract_reason"]`, export maps the status to `STALE`.
- `beliefs_type` comes from `metadata["beliefs_type"]`; otherwise it defaults to `DERIVED` for justified nodes and `OBSERVATION` for premises.
- `Depends on:` is built from all justification antecedents.
- `Unless:` is built from outlist nodes that are not marked as defeaters.
- `Defeated by:` is built from outlist nodes whose metadata has `defeats_node == node.id`, with `defeater_type` and optional `defeat_reason_type`.

### 3. Stale metadata handling

Stale handling is split between export-time status mapping and import-time retraction.

Export logic in `reasons/export_markdown.py`:

- A node is exported as `[STALE]` only when:
  - `node.truth_value == "OUT"`, and
  - `metadata["stale_reason"]` or `metadata["retract_reason"]` is present.
- The actual bullet is written as:

```markdown
- Stale reason: <retract_reason or stale_reason>
```

- It also emits `- Superseded by: ...` when `metadata["superseded_by"]` exists.

Import behavior is only partially observed. `tests/test_roundtrip.py::test_stale_node_round_trip` confirms that:

1. A node is retracted with a reason.
2. Export produces `[STALE]` and `Stale reason: ...`.
3. Import returns `claims_retracted == 1`.
4. The imported node has `truth_value == "OUT"`.

However, the test does **not** assert whether the importer preserves the reason as `retract_reason` or `stale_reason`, nor whether it preserves `superseded_by`. The underlying `Network.retract` implementation in `reasons/network.py` sets:

- `node.metadata["_retracted"] = True`
- `node.retracted_at`
- `node.metadata["retract_reason"] = reason` when a reason is provided

So the importer likely reconstructs a retraction rather than a special persistent `STALE` truth value, but exact metadata preservation is not shown.

### 4. `duplicate_of` handling

Export support is explicit:

- `reasons/export_markdown.py:106-107` writes:

```markdown
- Duplicate of: <node.metadata["duplicate_of"]>
```

`duplicate_of` is created elsewhere by the deduplication API:

- `reasons/api.py:1267` sets `meta["duplicate_of"] = canonical_id`
- `reasons/build_wiki.py:62-63` consumes `duplicate_of` for wiki generation

But the provided `duplicate_of` search results contain **no match in `reasons/import_beliefs.py`**, and the observed round-trip tests do not test `duplicate_of`. Therefore:

- The exporter definitely writes `- Duplicate of: ...`.
- There is no observed evidence that the markdown importer parses this bullet back into `metadata["duplicate_of"]`.
- If the importer ignores unknown bullets, `duplicate_of` would be lost on markdown round-trip.
- If the importer generically stores unknown metadata bullets, it might survive, but that is not demonstrated by the observations.

This is a material gap in the observed data.

### 5. Nogood section export and parsing gap

Export format in `reasons/export_markdown.py`:

```markdown
## Nogoods

### <nogood-id>
- Discovered: ...
- Resolution: ...
- Affects: x, y
```

Only `Discovered` and `Resolution` are emitted when present. `Affects` is always emitted from `ng.nodes`.

The import pipeline supports a `nogoods_text` argument and imports `_normalize_nogoods_markdown` at `reasons/api.py:1950`. However, the observed round-trip test explicitly records a format mismatch:

`tests/test_roundtrip.py::test_nogoods_export_format` says:

> “Verify nogoods are exported — note: parse_nogoods expects `nogood-N: label` format but export writes `nogood-NNN` without colon. Full nogood round-trip via markdown requires a format fix (pre-existing gap).”

So the exported nogood heading:

```markdown
### nogood-a-b
```

does not match what the nogood markdown parser is documented to expect. The test only asserts the export contains `## Nogoods` and `Affects: x, y`; it does not assert successful import. Therefore exported nogood sections are **not reliably round-trip safe** based on the observations.

### 6. What is missing from the observations

To fully explain the parser, we would need:

- `reasons/import_beliefs.py` implementation of `parse_beliefs`
- `reasons/import_beliefs.py` implementation of `import_into_network`
- `_normalize_nogoods_markdown` and `_normalize_nogoods_json`
- Any code that parses `- Duplicate of: ...`
- Any code that preserves stale metadata such as `stale_reason`, `retract_reason`, or `superseded_by` during import
- Any tests for `Repos` import and `duplicate_of` markdown round-trip

## Topics to Explore

- [file] `reasons/import_beliefs.py` — Contains the actual parsing implementation for `parse_beliefs`, `import_into_network`, stale statuses, and metadata bullets.
- [function] `reasons/import_beliefs.py:parse_beliefs` — Parses claim sections into dictionaries; tests show it extracts `source_url` and `accepted_pr`, but the full bullet-handling logic is not observed.
- [function] `reasons/import_beliefs.py:_normalize_nogoods_markdown` — Normalizes nogood markdown before import; imported in `api.py:1950`, but its implementation and format assumptions are missing.
- [function] `reasons/network.py:retract` — Shows the retraction metadata keys (`_retracted`, `retracted_at`, `retract_reason`) that the importer must reconstruct.
- [general] `duplicate-of-roundtrip` — Search for and test import handling of `- Duplicate of:`; no importer occurrence appears in the provided `duplicate_of` search results.

## Beliefs

- `markdown-export-writes-duplicate-of-bullet` — `export_markdown` emits `- Duplicate of: <id>` when `metadata["duplicate_of"]` exists (`reasons/export_markdown.py:106-107`).
- `duplicate-of-parser-not-in-observations` — The provided `duplicate_of` search results contain no match in `reasons/import_beliefs.py`.
- `stale-status-requires-out-and-reason` — Export maps a node to `[STALE]` only when its truth value is `OUT` and it has `stale_reason` or `retract_reason`.
- `stale-export-imports-as-retraction` — `test_stale_node_round_trip` verifies that an exported `[STALE]` node with `Stale reason` imports as a retracted node with `truth_value == "OUT"`.
- `nogood-markdown-roundtrip-has-known-format-mismatch` — The observed test notes that `parse_nogoods` expects `nogood-N: label`, while export writes `### nogood-...` without a colon.
