---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Locate where `_retracted` is actually set (not in the observations); those are the remaining legacy write sites.

## Where `_retracted` is actually mutated

The important distinction is that `_retracted` is not a Python variable in most places — it is a metadata **dictionary key**:

```python
node.metadata["_retracted"] = True
node.metadata.pop("_retracted", None)
```

That is why the regex `_retracted\s*=` in `direct__retracted_assignments` only found counter variables like `beliefs_retracted = 0`; it does **not** catch `metadata["_retracted"] = True`.

From the provided observations, the actual write/clear sites are:

| File | Line | Mutation | Notes |
|---|---:|---|---|
| `reasons/import_agent.py` | 311 | `network.nodes[node_id].metadata["_retracted"] = True` | Direct set |
| `reasons/import_agent.py` | 412 | `node.metadata["_retracted"] = True` | Conditional set under `if not node.metadata.get("_retracted")` |
| `reasons/import_agent.py` | 426 | `node.metadata.pop("_retracted", None)` | Clears flag when remote says `IN` |
| `reasons/import_agent.py` | 467 | `network.nodes[node_id].metadata["_retracted"] = True` | Direct set; comment says “Set `_retracted` directly instead of using `retract_after`” |
| `reasons/import_agent.py` | 478 | `node.metadata["_retracted"] = True` | Conditional set under `elif not node.metadata.get("_retracted")` |
| `reasons/import_agent.py` | 485 | `node.metadata.pop("_retracted", None)` | Clear |
| `reasons/import_agent.py` | 498 | `node.metadata["_retracted"] = True` | Set while counting `beliefs_retracted` |
| `reasons/network.py` | 163 | `node.metadata["_retracted"] = True` | In-memory `Network` write site |
| `reasons/api.py` | 1637 | `node.metadata.pop("_retracted", None)` | Clear/removal site |
| `reasons/pg.py` | ~372 | `metadata["_retracted"] = True` | Inside `PgApi.retract_node` |
| `reasons/pg.py` | ~420 | `metadata.pop("_retracted", None)` | Inside `PgApi.assert_node` |

The PostgreSQL sites are visible in the `pg_source` excerpt even though they do not appear in the `all__retracted_occurrences` match list. In `PgApi.retract_node`, the method loads `metadata`, sets `metadata["_retracted"] = True`, then writes it back as JSONB. In `PgApi.assert_node`, it removes `_retracted` and resets `retracted_at = NULL`.

The SQLite storage layer does **not** write `_retracted` directly. `reasons/storage.py` persists the entire node metadata blob via `json.dumps(node.metadata)` in `Storage.save`, so `_retracted` survives persistence as part of metadata rather than as its own column. The `retracted_at` column/field is a separate concept.

So the remaining legacy write sites are concentrated in:

- `reasons/import_agent.py` — most of them, including both set and clear paths
- `reasons/network.py:163` — in-memory network mutation
- `reasons/api.py:1637` — clearing path
- `reasons/pg.py:retract_node` / `reasons/pg.py:assert_node` — PostgreSQL set/clear paths

The `claims_retracted`, `beliefs_retracted`, `retracted_at`, and CLI/API result keys are not writes to the metadata flag.

## Topics to Explore

- [file] `reasons/import_agent.py` — Contains the majority of remaining `_retracted` metadata writes; inspect each import/propagation branch to decide migration order.
- [function] `reasons/pg.py:retract_node` — PostgreSQL retraction path that sets `metadata["_retracted"]` and `retracted_at`; compare with `assert_node`.
- [file] `reasons/network.py` — Locate the method around line 163 that still sets `_retracted` in the in-memory graph.
- [file] `reasons/api.py` — Inspect the clearing site at line 1637 and whether it belongs to an import/assert flow.
- [general] `metadata-retracted-flag-migration` — Plan how to remove the `_retracted` metadata flag and rely on `truth_value`/`retracted_at` consistently.

## Beliefs

- `metadata-retracted-flag-is-written-in-import-agent` — Most direct writes to `node.metadata["_retracted"]` are in `reasons/import_agent.py` at lines 311, 412, 467, 478, and 498.
- `pg-retract-node-sets-retracted-metadata` — `PgApi.retract_node` sets `metadata["_retracted"] = True` in `reasons/pg.py` around line 372.
- `pg-assert-node-clears-retracted-metadata` — `PgApi.assert_node` removes `_retracted` from metadata in `reasons/pg.py` around line 420.
- `storage-persists-retracted-flag-via-metadata-json` — SQLite `Storage` does not write `_retracted` directly; it persists the flag through `json.dumps(node.metadata)`.
- `retracted-at-is-distinct-from-retracted-flag` — `retracted_at` is a separate node field/column and is not the same as the `metadata["_retracted"]` flag.
