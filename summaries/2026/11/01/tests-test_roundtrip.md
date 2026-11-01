---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_roundtrip.py

# `tests/test_roundtrip.py`

## 1. Purpose

This file owns the **round-trip fidelity contract** for the `ftl-reasons` belief network system. It verifies that the serialization pipeline `Network → Markdown → Network` is lossless: every field set on a node (source, source_url, source_type, accepted_pr, source_hash, date, justifications, outlists, truth values, frontmatter metadata) must survive an export-to-markdown and re-import cycle unchanged.

It is the integration-level safety net that catches regressions when either `export_markdown` or `import_into_network`/`parse_beliefs` adds or renames a field. Without this file, a developer could break the markdown format in one direction without noticing until someone tried to re-import a previously exported file.

## 2. Key Components

### Fixture & Helper

| Name | Contract |
|---|---|
| `db` (fixture) | Creates an isolated SQLite DB under `tmp_path`. Yields the path string. Each test gets a fresh database. |
| `_load_network(db_path)` | Opens `Storage`, calls `.load()`, closes the handle, returns a `Network` object. Avoids the test having to manage the Storage lifecycle in every test. |

### `TestSourceUrlRoundTrip`

Four tests verifying the `source_url` field specifically:

- **`test_export_includes_source_url`** — asserts the literal line `- Source URL: <url>` appears in the markdown output.
- **`test_source_url_round_trip`** — the full cycle: `api.add_node(...)` → `export_markdown` → `import_into_network` → `net2.nodes["n1"].source_url == url`.
- **`test_source_url_omitted_when_empty`** — negative case: if no `source_url` was set, the string `"Source URL"` must **not** appear in the markdown.
- **`test_parse_source_url`** — bypasses the DB entirely; feeds a raw markdown string to `parse_beliefs` and checks the parsed dict contains the correct key/value. This isolates the parser from the exporter.

### `TestAcceptedPrRoundTrip`

Seven tests for the `accepted_pr` metadata field (a GitHub PR URL attached to a belief):

- Add, export, round-trip, omit-when-empty, and raw-parse tests mirror the `source_url` pattern.
- **`test_show_displays_accepted_pr`** / **`test_show_omits_when_absent`** — exercise the CLI `cmd_show` path, asserting the field appears (or doesn't) in stdout via `capsys`.

### `TestFullRoundTrip`

The most critical class. Six end-to-end tests:

| Test | What it pins down |
|---|---|
| `test_all_fields_survive` | 3 nodes (two observations + one derived with justifications). Checks **every** field: `text`, `source`, `source_url`, `source_hash`, `date`, `metadata["source_type"]`, `metadata["accepted_pr"]`, justification `antecedents`. |
| `test_stale_node_round_trip` | A retracted node (`truth_value == "OUT"`) with a `stale reason` must survive and the import result must report `claims_retracted == 1`. |
| `test_unless_round_trip` | Defeater (`unless`) lists on justifications: `j.outlist == ["b"]`. |
| `test_nogoods_export_format` | Documents a **known gap**: nogoods are exported but the markdown format (`nogood-NNN` without colon) doesn't match what `parse_nogoods` expects (`nogood-N: label`). The test only asserts the export side. |
| `test_double_round_trip_stability` | **Idempotency**: `export(import(export(net)))` must produce identical markdown (ignoring `updated_at` lines which change per import). |
| `test_repos_preserved` / `test_frontmatter_round_trip` | Non-node metadata: the `## Repos` section and YAML frontmatter keys like `project_name`. |

## 3. Patterns

- **Round-trip property testing.** The dominant idiom is: *write → serialize → deserialize → assert equality*. This is the standard pattern for any serialization format test.
- **Layered test isolation.** Each field is tested at three levels: (a) the API write path, (b) the markdown output string, (c) the parser (`parse_beliefs`) fed a raw string. If one layer is broken, the others still pass, so the combined round-trip test narrows the failure.
- **Negative assertions for omission.** Empty/absent optional fields must not produce output lines. This prevents the markdown from accumulating `- Source URL: ` with a blank value.
- **Known-gap documentation as test.** `test_nogoods_export_format` deliberately only tests the export side and documents in its docstring why the full round-trip is broken. This is a "failing test as TODO" pattern — it passes today but signals incomplete coverage.
- **Temp-file DB via `tmp_path`.** Standard pytest fixture; no cross-test contamination.
- **`capsys` for CLI tests.** `cmd_show` writes to stdout; the test captures it with `capsys.readouterr()` rather than mocking `print`.

## 4. Dependencies

### This file imports

| Module | What's used |
|---|---|
| `reasons.api` | `init_db`, `add_node`, `show_node`, `retract_node`, `add_nogood` — the public write/query surface |
| `reasons.export_markdown` | `export_markdown` — the serializer under test |
| `reasons.import_beliefs` | `import_into_network`, `parse_beliefs` — the deserializers under test |
| `reasons.network` | `Network` — the in-memory data structure |
| `reasons.storage` | `Storage` — persistence layer (load/save) |
| `reasons.cli` | `cmd_show` — CLI handler (imported inside test methods, not at module level) |
| `argparse` | Builds `Namespace` objects to call `cmd_show` directly |

### Who depends on this file

No other module imports this test file. It is a leaf in the dependency graph, executed only by the test runner (`pytest`).

## 5. Flow

The canonical execution path per test (e.g., `test_source_url_round_trip`):

```
1. api.init_db(db_path)              # create SQLite schema
2. api.add_node("n1", ..., db_path)  # INSERT node row with all fields
3. Storage(db_path).load()           # SELECT → hydrate Network object in memory
4. export_markdown(net)              # Network → Markdown string
5. import_into_network(net2, md)     # Markdown string → new Network
6. assert net2.nodes["n1"].source_url == url
```

For `test_all_fields_survive`, steps 2–3 repeat for multiple nodes, and step 6 expands to ~15 field-level assertions across three nodes.

For the double-round-trip test, steps 4–5 are repeated: `export_markdown(net2)` produces `md2`, and the test asserts `md1 == md2` (after stripping `updated_at` lines, since `import_into_network` stamps a new timestamp).

## 6. Invariants

1. **Lossless serialization.** Every non-None, non-empty field set on a node must appear in the markdown and be recoverable by the importer. This is the core contract the entire file enforces.
2. **Omission of empty fields.** If a field is `None` or `""`, the corresponding `- Field: value` line must be absent from the markdown. This prevents ambiguity between "field not set" and "field set to empty string."
3. **Idempotent export.** `export(import(export(N)))` must equal `export(N)`. The only permitted difference is the `updated_at` timestamp (filtered out in `test_double_round_trip_stability`).
4. **Node identity preservation.** Node IDs (`"obs-1"`, `"derived-1"`) are the join key; they must be identical after round-trip.
5. **Justification structure fidelity.** Antecedent lists, outlists (defeaters), and the count of justifications per node must be preserved.
6. **Truth value preservation.** A retracted node (`"OUT"`) stays `"OUT"` after import; it is not silently revived to `"IN"`.
7. **Frontmatter and repos are first-class.** Network-level metadata (`net.meta`, `repos` dict) round-trips alongside node data.

## 7. Error Handling

This is a test file, so "error handling" means **how failures are surfaced**:

- **Assertions are the only error mechanism.** There are no `try/except` blocks. A mismatch raises `AssertionError` with pytest's diff output.
- **`test_nogoods_export_format`** is the one place a known error is *acknowledged but not raised*. The docstring explicitly states the format mismatch is a "pre-existing gap" and the test only validates the export side. This is a deliberate scoping decision — it prevents the test suite from failing on a known issue while still documenting it.
- **`_load_network`** does not handle missing DB files or schema errors; it would raise whatever `Storage.load()` raises. This is fine for a test file — a broken fixture is a test error, not something to swallow.
- **CLI tests** (`test_show_displays_accepted_pr`) use `capsys` to capture stdout. If `cmd_show` raises, the test fails with that exception. There is no mocking of `cmd_show`'s internals.

---

## Topics to Explore

- [function] `reasons/export_markdown.py:export_markdown` — The serializer under test. Understanding how it walks the network and emits markdown lines reveals exactly which fields are covered and which might be silently dropped.
- [function] `reasons/import_beliefs.py:parse_beliefs` — The markdown-to-dict parser. The `test_nogoods_export_format` gap lives here: the expected `nogood-N: label` format vs. what the exporter actually writes.
- [file] `reasons/network.py` — The `Network` and node data structures. Knowing the full field set (especially `metadata` dict vs. top-level attributes like `source_url`, `source_hash`) clarifies why some fields are in `metadata` and others are direct attributes.
- [file] `tests/test_import_beliefs.py` — Likely contains more granular parser tests (edge cases in markdown parsing, malformed input). Comparing it with this file shows the test-strategy split between unit-level (parser) and integration-level (round-trip).
- [general] `nogood-export-format-mismatch` — The docstring in `test_nogoods_export_format` references a format gap between exporter and parser. Tracing how nogoods are written vs. read will clarify whether this is a real bug or a deliberate simplification.

## Beliefs

- `round-trip-is-lossless` — Every non-empty field on a node (source, source_url, source_hash, date, source_type, accepted_pr, text, truth_value, justifications, outlists) must survive an `export_markdown` → `import_into_network` cycle with exact value equality.
- `empty-fields-produce-no-markdown-lines` — If an optional field is `None` or absent, the corresponding `- Field: value` line must not appear in the exported markdown; the importer must not synthesize a default.
- `double-round-trip-is-idempotent` — `export_markdown(import_into_network(export_markdown(net)))` produces byte-identical markdown to `export_markdown(net)`, except for `updated_at` timestamp lines.
- `accepted-pr-lives-in-metadata-dict` — The `accepted_pr` value is stored in `node.metadata["accepted_pr"]`, not as a top-level attribute like `source_url`, and the CLI `cmd_show` reads it from `node["metadata"]["accepted_pr"]`.
- `nogood-round-trip-is-known-broken` — The markdown exporter writes nogood entries in a format (`nogood-NNN` without colon) that `parse_nogoods` does not recognize (expects `nogood-N: label`), making full nogood round-trip via markdown currently impossible.
