---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_source_type.py

# `tests/test_source_type.py`

## 1. Purpose

This file is a focused **contract test suite** for the `source_type` metadata field on nodes. It verifies that the field is:

- validated and stored on creation (`api.add_node`),
- surfaced consistently through the CLI (`cmd_show`), the node listing API (`api.list_nodes`), and markdown export (`export_markdown`),
- parsed and preserved through the belief-import path (`parse_beliefs` / `import_into_network`),
- filled in automatically when a node is derived (`derive.apply_proposals`).

In short: it locks down the *allowlist* of source types, the *shape* of the field on every output surface (metadata dict vs. top-level key vs. `""` vs. omitted), and the *round-trip* behavior between export and import.

## 2. Key Components

### Fixture

- **`db(tmp_path)`** — Creates a fresh SQLite DB in a pytest `tmp_path`, calls `api.init_db(db_path=db_path)`, and returns the path. Every non-import test uses this for isolation.

### Test Classes

- **`TestAddNodeSourceType`** — Tests `api.add_node(..., source_type=...)`:
  - `test_stores_source_type`: `show_node` exposes it under `node["metadata"]["source_type"]`.
  - `test_all_valid_types`: enumerates the allowlist `["code", "document", "self-description", "derived"]` and asserts each round-trips.
  - `test_invalid_type_raises`: `pytest.raises(ValueError, match="Invalid source_type")`.
  - `test_empty_type_not_stored`: when omitted, the `source_type` key is **absent** from metadata (not `None`, not `""`).
  - `test_source_type_persists`: verifies survival through `api.export_network` (`result["nodes"]["n1"]["metadata"]["source_type"]`).

- **`TestShowSourceType`** — Tests `reasons.cli.cmd_show`.
  - Constructs `argparse.Namespace(node_id, db, visible_to, pg_conninfo, pg_project)` to invoke the CLI handler **without** the CLI parser.
  - Uses pytest's `capsys` to assert the line `"Source type: code"` appears when present and `"Source type"` does **not** appear when absent.

- **`TestListSourceType`** — Tests `api.list_nodes`.
  - `node["source_type"] == "code"` when set.
  - `node["source_type"] == ""` when absent. Note this is a **different shape** than `show_node` (top-level `""` vs. missing key).

- **`TestExportMarkdown`** — Tests `reasons.export_markdown.export_markdown`.
  - Loads a `Network` with `Storage(db)`, then asserts `"- Source type: code"` is present in the markdown, and absent when no source type.

- **`TestImportBeliefs`** — Tests `parse_beliefs` and `import_into_network`.
  - `test_parses_source_type`: parses `- Source type: code` from a claim block.
  - `test_missing_source_type`: yields `claims[0]["source_type"] == ""`.
  - `test_import_stores_source_type`: `import_into_network(net, text)` puts the value into `net.nodes["obs-1"].metadata`.
  - `test_roundtrip_export_import`: export a node to markdown, re-import into a **fresh** `Network`, confirm the source type survives.

- **`TestDeriveAutoSourceType`** — Tests `reasons.derive.apply_proposals`.
  - `apply_proposals(proposals, db_path=db)` returns a list of `(proposal, result)` tuples; the test unpacks `_, result = results[0]` and confirms `result` is a `dict`, then asserts the newly created node `d1` has `source_type == "derived"`.

## 3. Patterns

- **Feature-sliced test classes** — each class names one subsystem (add, show, list, export, import, derive) and clusters assertions for that surface.
- **Layering respected** — most tests go through the public `api` module or the storage/network objects, not the SQLite layer directly.
- **Local imports** — for components used by only one class (`cmd_show`, `Storage`, `apply_proposals`), the import is scoped inside the test to keep the module's top-level deps minimal.
- **Round-trip testing** — `test_roundtrip_export_import` asserts the *serialize → parse → import* path is lossless, which is the strongest contract for a metadata field.
- **CLI handler test via `argparse.Namespace`** — avoids spawning `subprocess` and exercises `cmd_show` as a function.
- **Fixture-scoped DB isolation** — `tmp_path` + `init_db` gives each test a clean database; no ordering dependencies between tests.

## 4. Dependencies

**Depends on (imports):**

- `pytest` (fixture, `raises`, `capsys`).
- `reasons.api` — `init_db`, `add_node`, `show_node`, `list_nodes`, `export_network`.
- `reasons.export_markdown.export_markdown` — markdown serializer.
- `reasons.import_beliefs.parse_beliefs` / `import_into_network` — markdown parser/importer.
- `reasons.network.Network` — in-memory graph container.
- Local: `reasons.cli.cmd_show`, `reasons.storage.Storage`, `reasons.derive.apply_proposals`.

**Imported by:** nothing — it's a leaf test module. But this file *defines contracts* that `api.py`, `cli.py`, `export_markdown.py`, `import_beliefs.py`, and `derive.py` must continue to honor.

**Notably unused imports at top:** `json`, `subprocess`, and `Path` are imported but never referenced. A maintainer can safely remove these (or future linters/CI may flag them).

## 5. Flow

Typical execution of one test:

1. **Setup**: `db` fixture creates `tmp_path/test.db`, calls `api.init_db(db_path=...)`, returns path.
2. **Seed**: `api.add_node("n1", "...", source_type="code", db_path=db)`.
3. **Exercise one surface**:
   - `api.show_node(...)` → inspect `node["metadata"]["source_type"]`.
   - `api.list_nodes(...)` → inspect `node["source_type"]`.
   - `cmd_show(Namespace(...))` → inspect `capsys.readouterr().out`.
   - `Storage(db).load()` → `export_markdown(net)` → inspect returned string.
   - `parse_beliefs(text)` / `import_into_network(net, text)` → inspect return dict and node metadata.
4. **Assert** presence, absence, or error.

The derive test differs slightly: it seeds premises `p1`, `p2`, calls `apply_proposals([{...antecedents: ["p1","p2"]...}])`, then re-queries via `api.show_node("d1")` to confirm `source_type="derived"`.

## 6. Invariants

- **Allowlist**: source types are exactly `code`, `document`, `self-description`, `derived`. No other string is a valid `source_type`.
- **Validation error contract**: invalid input raises `ValueError` whose message contains the substring `"Invalid source_type"`.
- **Representation asymmetry** (intentional, tested):
  - `show_node(...)["metadata"]` **omits** the key when unset (`"source_type" not in metadata`).
  - `list_nodes(...)["nodes"][i]` returns `""` when unset.
  - CLI `cmd_show` **omits** the line entirely when unset.
  - Markdown export **omits** the `- Source type:` line when unset.
- **Round-trip stability**: `export_markdown` emits `- Source type: <value>`; `parse_beliefs` reads it back into `source_type`; `import_into_network` writes it to node metadata. A node added via `api.add_node` and re-imported through this pipeline ends up byte-equivalent in source_type.
- **Derived nodes are auto-stamped**: `apply_proposals` creates derivation nodes with `source_type="derived"` — callers do not need to pass it.
- **`parse_beliefs` default**: missing source type is `""` (not `None`, not a KeyError).

## 7. Error Handling

- The suite's only explicit error-path test is `test_invalid_type_raises`, which pins the **exception type** (`ValueError`) and **message substring** (`"Invalid source_type"`) that `api.add_node` must raise. Changing that message string will break this test.
- No test here exercises DB-level failures, corrupted imports, or export failures — those are the responsibility of the corresponding unit tests (`test_storage.py`, `test_import_beliefs.py`, `test_export_markdown.py`).
- The `test_derive_sets_derived_type` test tolerates whatever `apply_proposals` returns as long as `result` is a `dict`; it does not assert on error fields, so a proposal that fails to apply would surface as a missing `d1` node instead.

## Topics to Explore

- [file] `reasons/api.py` — Where `add_node` validates `source_type` and where `show_node` / `list_nodes` shape the asymmetrical output (`metadata` key vs. top-level `""`).
- [function] `reasons/import_beliefs.py:parse_beliefs` — How the `- Source type:` line is recognized, and why missing yields `""` instead of `None`.
- [file] `reasons/export_markdown.py` — The exact line format `- Source type: <value>` and the omission logic when metadata is missing.
- [function] `reasons/derive.py:apply_proposals` — How source type `"derived"` is stamped on generated nodes and what the `(proposal, result)` tuple shape guarantees.
- [general] `metadata-schema-evolution` — How `source_type` fits into the broader node metadata model, and whether the allowlist is expected to grow or migrate.

## Beliefs

- `source-type-allowlist` — `api.add_node` accepts only `code`, `document`, `self-description`, and `derived` as `source_type`.
- `add-node-rejects-invalid-source-type` — Invalid `source_type` values raise `ValueError` with a message containing `"Invalid source_type"`.
- `source-type-omitted-when-absent` — When no source type is supplied, `show_node` returns metadata with no `source_type` key while `list_nodes` returns an empty string.
- `export-markdown-emits-source-type` — `export_markdown` writes the line `- Source type: <value>` only for nodes whose metadata contains `source_type`.
- `derive-auto-assigns-derived-source-type` — `apply_proposals` stamps generated derivation nodes with `source_type="derived"` without caller input.
