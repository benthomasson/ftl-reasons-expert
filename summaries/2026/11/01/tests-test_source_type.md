---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_source_type.py

# `tests/test_source_type.py`

## Purpose

This file is the **integration test suite for the `source_type` metadata field** across the entire `reasons` belief-network system. It verifies that a node's provenance classification (`"code"`, `"document"`, `"self-description"`, or `"derived"`) is correctly:

- Stored and validated at the API layer
- Displayed in CLI output (`show` and `list` commands)
- Exported to markdown
- Parsed back during belief import
- Auto-assigned by the derivation engine

It owns the contract that `source_type` is a first-class, round-trip-safe metadata field — not just an afterthought.

## Key Components

| Component | Responsibility |
|-----------|---------------|
| `db` fixture | Creates a fresh SQLite DB per test via `tmp_path` + `api.init_db`. Provides isolation. |
| `TestAddNodeSourceType` | Validates write-path: storage, enum enforcement, persistence across export. |
| `TestShowSourceType` | Validates the CLI `show` command renders (or omits) the field. |
| `TestListSourceType` | Validates the `list_nodes` API surfaces `source_type` as a flat key. |
| `TestExportMarkdown` | Validates `export_markdown` emits `- Source type: <value>` in markdown. |
| `TestImportBeliefs` | Validates `parse_beliefs` extracts the field from markdown, and `import_into_network` stores it. Includes a **round-trip** test. |
| `TestDeriveAutoSourceType` | Validates that `apply_proposals` (the derivation engine) stamps `source_type="derived"` automatically. |

### Valid source types (the enum)

The accepted values are exactly: `"code"`, `"document"`, `"self-description"`, `"derived"`. Any other string triggers a `ValueError`.

## Patterns

- **Pytest fixtures for isolation** — `db` uses `tmp_path` so every test gets a clean database; no cross-test contamination.
- **Table-driven test** — `test_all_valid_types` iterates over all four valid values in a loop, asserting each round-trips.
- **Round-trip / property-based testing** — `test_roundtrip_export_import` exports to markdown, re-imports, and asserts the metadata survives the cycle. This catches asymmetries between the export and import parsers.
- **CLI testing via `capsys`** — `TestShowSourceType` calls `cmd_show` with a synthetic `argparse.Namespace` and captures stdout, verifying the human-facing format string.
- **Positive + negative pairs** — Every feature is tested with the field present *and* absent (e.g., `test_exports_source_type` / `test_omits_when_absent`).
- **Lazy imports for CLI/Storage** — `cmd_show` and `Storage` are imported inside test methods rather than at module top, likely to avoid circular-import issues at collection time.

## Dependencies

### This file imports (direct):

```
reasons.api                  → add_node, show_node, list_nodes, export_network, init_db
reasons.export_markdown      → export_markdown
reasons.import_beliefs       → parse_beliefs, import_into_network
reasons.network              → Network (container for nodes)
reasons.cli                  → cmd_show
reasons.storage              → Storage (DB → Network loader)
reasons.derive               → apply_proposals
pytest, argparse, json, subprocess, pathlib.Path
```

> Note: `json`, `subprocess`, and `Path` are imported at the top but **never referenced** in the visible test bodies — likely dead imports from a refactor.

### What depends on this file:

Nothing in production code imports a test module. This file is consumed by the test runner (`pytest`) and CI (`.github/workflows`).

## Flow

Each test class follows the same skeleton:

```
1. Fixture: api.init_db(tmp_path)  →  fresh SQLite file
2. Arrange: api.add_node(...) or Network() + manual setup
3. Act:     the operation under test (show, list, export, import, derive)
4. Assert:  inspect the returned structure / captured stdout
```

The **round-trip test** is the most complex flow:

```
api.add_node (source_type="code")
  → Storage(db).load() → Network
    → export_markdown(net) → markdown string
      → import_into_network(Network(), markdown) → Network
        → assert node.metadata["source_type"] == "code"
```

The **derive test** exercises a different entry point:

```
api.add_node("p1") ; api.add_node("p2")
  → apply_proposals([{id:"d1", antecedents:["p1","p2"], ...}], db_path=db)
    → api.show_node("d1") → assert metadata.source_type == "derived"
```

This proves the derivation engine injects the type automatically — the caller never specifies it.

## Invariants

1. **Closed enum**: Only the four strings `["code", "document", "self-description", "derived"]` are valid. Anything else is rejected at write time.
2. **Optional field**: Omitting `source_type` is legal; the key simply won't appear in `node["metadata"]`. No default is injected.
3. **Derivation stamps automatically**: Nodes created via `apply_proposals` always receive `source_type="derived"` regardless of what the proposer specifies.
4. **Round-trip fidelity**: `export_markdown` → `import_into_network` must preserve the field exactly.
5. **List API flattens**: `list_nodes` exposes `source_type` as a top-level key on each node dict (not nested under `metadata`), defaulting to `""` when absent.

## Error Handling

| Scenario | Behavior |
|----------|----------|
| `source_type="unknown"` passed to `api.add_node` | Raises `ValueError` with message matching `"Invalid source_type"` — **fails fast at write time**. |
| `source_type` omitted | No error. The key is absent from metadata. Downstream consumers (CLI, markdown) must handle the missing key. |
| `source_type=""` (empty string) | Treated as "not provided" — not stored. (Implicit from `test_empty_type_not_stored`.) |

The validation is **server-side** (in the API layer), not deferred to the UI or serializer. The test confirms the `ValueError` is raised synchronously during `add_node`, before any DB write.

---

## Topics to Explore

- [file] `reasons/api.py` — Contains `add_node`, `show_node`, `list_nodes`; this is where the `source_type` enum validation lives and where the `ValueError` is raised.
- [file] `reasons/import_beliefs.py` — The `parse_beliefs` function that extracts `- Source type: <value>` from markdown bullet lists; understanding its regex/parsing strategy is critical for the round-trip invariant.
- [function] `reasons/derive.py:apply_proposals` — The code path that automatically stamps `source_type="derived"`; useful to confirm *where* in the pipeline the assignment happens (before or after validation).
- [file] `reasons/export_markdown.py` — The serializer that decides the exact markdown format (`- Source type: code`) consumed by the import parser; any format drift here breaks the round-trip test.
- [general] `source-type-enum-location` — The valid set `["code", "document", "self-description", "derived"]` is presumably defined as a constant somewhere; knowing its canonical location helps when adding a new type.

## Beliefs

- `add-node-rejects-invalid-source-type` — `api.add_node` raises `ValueError` for any `source_type` not in the four-value enum; it does not silently coerce or store the raw string.
- `derive-automatically-sets-source-type-derived` — `apply_proposals` unconditionally stamps `metadata["source_type"] = "derived"` on newly created nodes, independent of the caller's input.
- `list-nodes-flattens-source-type-to-empty-string` — When a node has no `source_type`, `api.list_nodes` returns `""` (not `None`, not a missing key) for the `source_type` field on that node's dict.
- `export-import-roundtrip-preserves-source-type` — `export_markdown` → `import_into_network` is lossless for the `source_type` field; the markdown bullet `- Source type: X` is the sole wire format between the two.
- `source-type-is-optional-in-node-metadata` — The `source_type` key is absent (not `null`, not `""`) in `node["metadata"]` when never set; consumers must check for key existence rather than truthiness.
