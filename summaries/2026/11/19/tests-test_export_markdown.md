---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_export_markdown.py

## Purpose

`tests/test_export_markdown.py` is the pytest unit-test contract for `reasons.export_markdown.export_markdown`. Its job is to verify that an in-memory `Network` of beliefs is serialized into the expected Markdown format: registry header, claims section, node headings, metadata lines, dependency/defeater rendering, ordering, and nogoods.

It owns no production logic. It documents and locks down the public behavior of the Markdown exporter and, indirectly, the `Network` and `Justification` data contracts used by that exporter.

## Key Components

The file defines one test class, `TestExportMarkdown`, with 12 independent test methods and no fixtures, helpers, setup/teardown, or parametrization.

Imports:

- `Justification` from `reasons` — used to construct derivation metadata, including `type`, `antecedents`, and `outlist`.
- `Network` from `reasons.network` — the in-memory belief graph used as exporter input.
- `export_markdown` from `reasons.export_markdown` — the function under test.

The tests exercise these `Network` methods:

- `add_node(...)`
- `retract(...)`
- `add_nogood(...)`
- `recompute_all()`

| Test | Contract being enforced |
|---|---|
| `test_empty_network` | Empty export still contains `# Belief Registry` and `## Claims`. |
| `test_premise` | A premise renders as `### a [IN] OBSERVATION`, includes its content, and emits `- Source:`, `- Source hash:`, and `- Date:` lines. |
| `test_derived_node` | A node with `Justification(type="SL", antecedents=["a"])` renders as `### b [IN] DERIVED` and includes `- Depends on: a`. |
| `test_retracted_node` | `net.retract("a")` changes the heading to `### a [OUT] OBSERVATION`. |
| `test_stale_metadata` | Metadata keys `stale_reason` and `superseded_by` render as `[STALE]`, `- Stale reason: superseded`, and `- Superseded by: b`; in this scenario `[STALE]` takes precedence over `[OUT]`. |
| `test_in_before_out` | The `IN` node text appears before the `OUT` node text using `md.index(...)`. |
| `test_nogoods_section` | `add_nogood(["a", "b"])` produces a `## Nogoods` section and a `nogood-a-b` identifier. |
| `test_beliefs_type_preserved` | `metadata={"beliefs_type": "WARNING"}` renders as `### a [IN] WARNING`, so metadata overrides the default node type label. |
| `test_multiple_antecedents` | Antecedents `["a", "b"]` render as `- Depends on: a, b`. |
| `test_defeater_in_outlist` | An outlist entry whose node has defeater metadata renders as `- Defeated by: defeater-1 (invalid-inference)` and suppresses `Unless`. |
| `test_mixed_outlist_defeater_and_regular` | A mixed outlist is split: defeater metadata renders via `- Defeated by:`, regular outlist nodes render via `- Unless:`. |
| `test_defeater_with_reason_type` | `defeat_reason_type` is appended inside parentheses: `- Defeated by: defeater-1 (invalid-inference, unsupported-conjunct)`. |

## Patterns

- **Arrange-Act-Assert**: each test creates a fresh `Network`, mutates it, calls `export_markdown(net)`, and asserts against the returned Markdown string.
- **Isolated in-memory tests**: no database, file system, network, or fixture state is used.
- **Substring assertions rather than full snapshots**: tests assert exact important fragments like `"### a [IN] OBSERVATION"` and `"- Depends on: a, b"`. This allows surrounding formatting to evolve as long as the contractual fragments remain.
- **Metadata-driven behavior**: the exporter is expected to respond to metadata keys such as `stale_reason`, `superseded_by`, `beliefs_type`, `defeater_type`, `defeats_node`, and `defeat_reason_type`.
- **Ordering assertion via `md.index`**: `test_in_before_out` verifies relative position in the output string. Note that because the node IDs are `a` and `b`, this test would also pass if the exporter simply sorted by ID; it does not fully isolate status grouping from ID ordering.
- **Positive-path testing only**: the file does not test malformed metadata, missing referenced outlist nodes, duplicate IDs, or exporter exceptions.

## Dependencies

This test module depends on:

- `reasons.Justification`
- `reasons.network.Network`
- `reasons.export_markdown.export_markdown`

It is discovered and executed by pytest. No production code imports this test file.

The repository contains both `reasons/` and `reasons_lib/` with similarly named modules. These tests import from `reasons`, not `reasons_lib`, so they directly cover `reasons/export_markdown.py` and related `reasons` models only. If `reasons_lib` is maintained as a mirrored or published package, synchronization must be checked separately.

## Flow

Each test follows the same general execution path:

1. Create a new `Network`.
2. Add one or more nodes with `add_node(...)`, optionally passing:
   - `source`
   - `source_hash`
   - `date`
   - `justifications=[Justification(...)]`
   - `metadata={...}`
3. Optionally mutate state with `retract(...)`, `add_nogood(...)`, or `recompute_all()`.
4. Call `export_markdown(net)`.
5. Assert that specific Markdown fragments are present and, in one case, ordered correctly.

The exporter behavior implied by the assertions is roughly:

- Emit a top-level `# Belief Registry`.
- Emit a `## Claims` section.
- Render each node under a heading like `### <id> [<status>] <type>`.
- Render node content below the heading.
- Render metadata lines such as source, source hash, date, stale reason, and superseded-by.
- Render derived dependencies as `- Depends on: ...`.
- Inspect justification `outlist` entries:
  - If the referenced node looks like a defeater, render `- Defeated by: <id> (<defeater_type>[, <defeat_reason_type>])`.
  - Otherwise render `- Unless: <id>`.
- Emit a `## Nogoods` section when nogoods exist.

The defeater-related tests call `net.recompute_all()` before export, so those scenarios are coupled to the network’s computed state as well as the exporter formatting logic.

## Invariants

The tests enforce these output rules:

- Empty output still contains:
  - `# Belief Registry`
  - `## Claims`
- Claim headings use the format:
  - `### <id> [<STATUS>] <TYPE>`
- Status values observed in this file:
  - `[IN]`
  - `[OUT]`
  - `[STALE]`
- Type values observed in this file:
  - `OBSERVATION` for a premise with no overriding metadata
  - `DERIVED` for a node with justifications
  - `WARNING` when `metadata={"beliefs_type": "WARNING"}`
- Metadata lines use exact labels:
  - `- Source: ...`
  - `- Source hash: ...`
  - `- Date: ...`
  - `- Stale reason: ...`
  - `- Superseded by: ...`
- Multiple antecedents are comma-space joined:
  - `- Depends on: a, b`
- A stale node with `stale_reason` renders `[STALE]` rather than `[OUT]` in the tested scenario.
- Defeater outlist entries render as:
  - `- Defeated by: defeater-1 (invalid-inference)`
  - `- Defeated by: defeater-1 (invalid-inference, unsupported-conjunct)`
- Regular outlist entries render as:
  - `- Unless: blocker`
- When an outlist entry is a defeater, it should not also appear as an `Unless` entry in the simple defeater test.
- For `add_nogood(["a", "b"])`, the output contains `nogood-a-b`.

## Error Handling

There is no explicit error-handling coverage in this file:

- No `try/except`.
- No `pytest.raises`.
- No assertions about invalid node IDs, missing outlist targets, malformed metadata, or exporter exceptions.

If `export_markdown` raises an exception, the test will error. If an expected Markdown fragment is absent, pytest reports an assertion failure. This file is a formatting contract test, not a validation or failure-mode test.

## Topics to Explore

- [file] `reasons/export_markdown.py` — The production serializer under test; inspect how it groups statuses, formats headings, splits outlist entries, and emits nogoods.
- [function] `reasons/network.py:Network.recompute_all` — Called before export in the defeater tests; explains how derived status and outlist defection interact with the network model.
- [function] `reasons/export_markdown.py:export_markdown` — Main serialization contract; trace section ordering and metadata-driven rendering.
- [general] `defeater-metadata-model` — How `defeater_type`, `defeat_reason_type`, and `defeats_node` are produced and consumed across the codebase.
- [repo] `reasons-vs-reasons_lib-sync` — Tests target `reasons`, but the repository also contains mirrored `reasons_lib` modules; confirm whether changes must be mirrored.

## Beliefs

- `export-markdown-tests-target-reasons-package` — The test module imports `Justification`, `Network`, and `export_markdown` from the `reasons` package, not from `reasons_lib`.
- `export-markdown-heading-encodes-status-and-type` — Claim headings are asserted as `### <id> [<STATUS>] <TYPE>` with statuses `IN`, `OUT`, or `STALE` and types such as `OBSERVATION`, `DERIVED`, or `WARNING`.
- `export-markdown-splits-outlist-defeaters-and-blockers` — Outlist nodes with defeater metadata render via `- Defeated by:`, while outlist nodes without defeater metadata render via `- Unless:`.
- `export-markdown-includes-optional-node-metadata-lines` — The exporter is expected to emit metadata lines such as `- Source:`, `- Source hash:`, `- Date:`, `- Stale reason:`, and `- Superseded by:` when present.
- `defeater-export-tests-call-recompute-all` — All three defeater/outlist tests call `Network.recompute_all()` before invoking `export_markdown`.
