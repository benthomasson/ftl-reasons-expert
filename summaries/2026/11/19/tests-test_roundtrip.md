---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_roundtrip.py

# Explanation of `tests/test_roundtrip.py`

## 1. Purpose

This file is an **integration-style contract test for the markdown serialization boundary**. Its sole responsibility is to prove that the pipeline

```
add nodes to SQLite  →  Storage.load()  →  export_markdown()  →  import_into_network()  →  Network
```

is **lossless for the fields the project cares about**. Because the human-readable markdown is the artifact that gets committed to git, handed to LLMs, and read back by the importer, any field that silently drops on the way in or out is a data-loss bug. This file pins that contract down field-by-field.

It is **not** a unit test of export or import in isolation — it exercises them together (or via `parse_beliefs` for parser-only checks), which is why the module is named `test_roundtrip`.

## 2. Key Components

### Fixtures and helpers

- **`db(tmp_path)`** — creates a temp SQLite file under `tmp_path`, calls `api.init_db(db_path=...)`, returns the path string. This gives each test an isolated DB.
- **`_load_network(db_path)`** — opens `Storage(db_path)`, calls `.load()` to produce a `Network`, then `.close()`. Used everywhere a fresh snapshot of the persisted state is needed.

### Test classes

**`TestSourceUrlRoundTrip`** — coverage of the `source_url` field:
- `test_export_includes_source_url` — asserting `"- Source URL: <url>"` appears in exported markdown.
- `test_source_url_round_trip` — full export→import→compare on `net2.nodes["n1"].source_url`.
- `test_source_url_omitted_when_empty` — empty field must **not** produce the line.
- `test_parse_source_url` — `parse_beliefs` directly on hand-written markdown, asserting `claims[0]["source_url"]`.

**`TestAcceptedPrRoundTrip`** — coverage of `accepted_pr`, which is stored under `node.metadata`:
- `test_add_with_accepted_pr` — verifies `api.add_node(..., accepted_pr=...)` stores it in `show_node(...)["metadata"]["accepted_pr"]`.
- `test_export_includes_accepted_pr` / `test_accepted_pr_round_trip` / `test_accepted_pr_omitted_when_empty` / `test_parse_accepted_pr` — the same four cases as `source_url`.
- `test_show_displays_accepted_pr` / `test_show_omits_when_absent` — drive the actual CLI `cmd_show` with an `argparse.Namespace(node_id, db, visible_to, pg_conninfo, pg_project)` and use `capsys` to assert `"Accepted PR: ..."` presence/absence in stdout. Note the lazy `from reasons.cli import cmd_show` import inside the test bodies to avoid pulling CLI dependencies into every test module.

**`TestFullRoundTrip`** — end-to-end tests:
- `test_all_fields_survive` — the flagship test. Adds `obs-1` with `source`, `source_url`, `source_type`, `accepted_pr`; **manually** sets `source_hash` and `date` on `net.nodes["obs-1"]` and re-saves, because "source_hash and date aren't API params". Then adds `obs-2` and `derived-1` (with `sl="obs-1,obs-2"`). Exports, imports, asserts `result["claims_imported"] == 3` and checks every field plus justification antecedents (`set(d1.justifications[0].antecedents) == {"obs-1", "obs-2"}`).
- `test_stale_node_round_trip` — `api.retract_node(...)` then export. Asserts `"[STALE]"` and `"Stale reason: ..."` markers appear, and importing sets `truth_value == "OUT"` with `result["claims_retracted"] == 1`.
- `test_unless_round_trip` — `api.add_node("c", ..., sl="a", unless="b")`. Exports `"- Unless: b"`, imports back to `justifications[0].antecedents == ["a"]` and `outlist == ["b"]`.
- `test_nogoods_export_format` — **regression-documenting test**. Adds a nogood and only asserts the export format (`"## Nogoods"`, `"Affects: x, y"`). The docstring explicitly names a pre-existing gap: *`parse_nogoods` expects `'nogood-N: label'` format but export writes `'nogood-NNN'` without a colon* — so this test stops short of round-tripping.
- `test_double_round_trip_stability` — idempotency check: `md1 == md2` after one import/export cycle, with lines starting with `updated_at` filtered out (that line embeds a timestamp that will differ).
- `test_repos_preserved` — calls `export_markdown(net, repos={"ftl-reasons": "/Users/ben/git/ftl-reasons"})` and asserts the `## Repos` section.
- `test_frontmatter_round_trip` — sets `net.meta["project_name"]`, asserts `'project_name: "test-project"'` (note: quoted) and that `net2.meta.get("project_name")` equals it after import.

## 3. Patterns

- **Arrange–Act–Assert with a round-trip identity**: nearly every test follows `add → load → export → import → assert`, matching the file name.
- **Fixture isolation via `tmp_path`** — no shared state; each test opens its own DB.
- **Lazy imports** — `cmd_show` and `argparse` are imported *inside* test methods to keep the module's import-time cost low and avoid CLI-only dependencies for the round-trip tests.
- **Direct model mutation as an escape hatch** — `test_all_fields_survive` sets `net.nodes["obs-1"].source_hash` and `.date` directly because `api.add_node` doesn't expose them; a comment documents this.
- **Documenting a known gap in a test** — `test_nogoods_export_format` intentionally does *not* assert round-trip, and the docstring names the reason so future maintainers don't "fix" the wrong side.
- **CLI smoke via `capsys`** — `test_show_*` uses `capsys.readouterr().out` rather than mocking.

## 4. Dependencies

Imports (module-level):
- `pytest`
- `reasons.api` — `init_db`, `add_node`, `retract_node`, `add_nogood`, `show_node`
- `reasons.export_markdown.export_markdown`
- `reasons.import_beliefs.import_into_network`, `parse_beliefs`
- `reasons.network.Network`
- `reasons.storage.Storage`

Imports (local to tests):
- `reasons.cli.cmd_show`, `argparse` — used only in `test_show_*`

Imported by: nothing — this is a leaf test module under `tests/`.

## 5. Flow

The dominant data flow:

1. `db` fixture creates a temp file and `api.init_db`s it.
2. Test calls `api.add_node(...)` / `api.retract_node(...)` / `api.add_nogood(...)`, which write to SQLite.
3. `_load_network(db)` reads a fresh `Network` snapshot.
4. `export_markdown(net [, repos=...])` produces the markdown string.
5. Either:
   - assertions on substrings (export contract), or
   - `parse_beliefs(md)` returns a list of dicts (parser contract, isolated), or
   - `import_into_network(Network(), md)` builds a new `Network` and returns a result dict (e.g. `{"claims_imported": 3, "claims_retracted": 1}`) — then assertions on the new network's nodes.

`test_double_round_trip_stability` runs steps 4→5 twice and compares filtered lines.

## 6. Invariants

- **Omitted-when-empty**: `source_url` and `accepted_pr` must not appear as lines in exported markdown when unset. This prevents noisy templates.
- **Metadata preservation**: `source_type` and `accepted_pr` round-trip through `node.metadata`, not through top-level attributes.
- **Truth-value preservation**: retraction survives export as `[STALE]` / `Stale reason:` and re-imports as `truth_value == "OUT"`.
- **Justification structure**: `sl` round-trips to `antecedents`; `unless` round-trips to `outlist`.
- **Stability**: `export(import(export(net))) == export(net)` modulo the `updated_at` timestamp line.
- **Frontmatter fidelity**: `net.meta` keys survive the YAML-ish frontmatter round-trip.
- **Repos section**: `repos=` kwarg on `export_markdown` maps to a `## Repos` block.

## 7. Error Handling

There is no explicit `try/except` in the file. Failure modes are localized to assertions:
- Missing field in export → substring assertion fails.
- Missing field in import → attribute equality assertion fails.
- Counts mismatch → `assert result["claims_imported"] == 3` fails.

The only "error-adjacent" logic is in `test_nogoods_export_format`, where the author **declines to test the round-trip** and instead documents the format mismatch in the docstring. This is a deliberate non-assertion that prevents the test suite from being red while still guarding the exporter half of the contract.

## Topics to Explore

- [file] `reasons/export_markdown.py` — Understand the exact markdown shape produced (frontmatter, `## Claims`, `## Repos`, `## Nogoods`) and which node fields are serialized.
- [file] `reasons/import_beliefs.py` — Read `parse_beliefs` / `import_into_network` to see the inverse parser and the result-dict contract (`claims_imported`, `claims_retracted`).
- [function] `reasons/export_markdown.py:export_markdown` — The `repos=None` kwarg and the `updated_at` emission that makes `test_double_round_trip_stability` filter lines.
- [general] `nogood-format-gap` — Trace `parse_nogoods` vs. export and confirm the `'nogood-N: label'` vs `'nogood-NNN'` mismatch called out in `test_nogoods_export_format`.
- [file] `reasons/network.py` — The `Network`/`Node`/`Justification` model since tests poke attributes like `source_hash`, `date`, `metadata`, `justifications[0].antecedents/outlist`, and `truth_value` directly.

## Beliefs

- `export-import-round-trip-preserves-metadata` — A node's `source_type` and `accepted_pr` stored in `node.metadata` survive `export_markdown → import_into_network` unchanged.
- `empty-optional-fields-omitted-from-export` — When `source_url` or `accepted_pr` is unset, `export_markdown` omits the corresponding markdown line entirely.
- `api-add-node-rejects-source-hash-and-date` — `api.add_node` does not accept `source_hash` or `date`; those must be set directly on the `Network` node and re-saved via `Storage.save`.
- `unless-round-trips-to-outlist` — An `unless` clause on a node exports as `- Unless: <id>` and re-imports as `justifications[0].outlist == [<id>]`.
- `retracted-nodes-export-as-STALE-and-import-as-OUT` — `api.retract_node` produces `[STALE]` + `Stale reason:` in markdown, and `import_into_network` restores `truth_value == "OUT"` with a `claims_retracted` count.
- `double-export-is-stable-modulo-updated-at` — Two successive export/import cycles yield identical markdown except for lines beginning with `updated_at`.
- `nogood-markdown-round-trip-is-not-implemented` — `export_markdown` emits `nogood-NNN` without a colon while `parse_nogoods` expects `nogood-N: label`, so nogoods are export-only via markdown today.
