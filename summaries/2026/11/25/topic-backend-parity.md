---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: The social contract that SQLite and Postgres methods return equivalent dict shapes; worth auditing whether `PgApi` methods actually mirror the SQLite branches for e.g. `show_node`, `summarize`, `import_json`.

## The social contract around equivalent dict shapes

The implied contract is that the public API is backend-agnostic: callers use the same function names and expect the same return dict schema whether the implementation talks to SQLite or Postgres. The Postgres path is selected through dispatch code and `PgApi`:

- `PgApi` is defined at `reasons/pg.py:98`.
- `show_node` exists as the SQLite-facing API function at `reasons/api.py:870` and as `PgApi.show_node` at `reasons/pg.py:678`.
- `summarize` exists at `reasons/api.py:1141`, with core network logic at `reasons/network.py:768`, and as `PgApi.summarize` at `reasons/pg.py:1726`.
- `import_json` exists at `reasons/api.py:2104` and as `PgApi.import_json` at `reasons/pg.py:2236`.

That naming overlap strongly suggests a social contract: CLI/API consumers should not need to branch on backend when reading returned dicts.

### What the observed tests actually prove

The tests show intent, but not full parity.

`tests/test_pg_dispatch.py` verifies that calls such as `import_json` go through `_pg_dispatch` and that `PgApi.import_json` is called with parsed data:

```python
mock_pg.import_json.assert_called_once_with(data=data)
assert result["nodes_imported"] == 1
```

But this test uses a `MagicMock` for `PgApi.import_json`; it does not prove the real `PgApi.import_json` at `reasons/pg.py:2236` returns the same shape as the SQLite `api.import_json` branch.

`tests/test_import_json.py` verifies SQLite behavior and expects keys like `nodes_imported` and `nogoods_imported`. It also uses `api.show_node(...)` and expects fields such as `source`, `truth_value`, `justifications`, and `metadata`.

`tests/test_pg.py::TestShowNode` verifies that `PgApi.show_node` returns at least:

- `id`
- `truth_value`
- `justifications`
- `dependents`

It also tests metadata fields such as `access_tags` and `retract_reason`, and access-control behavior. But the observed snippets do not show a direct comparison between SQLite `show_node` and `PgApi.show_node` full key sets.

`tests/test_summarize.py::TestSummarizeAPI` verifies SQLite `api.summarize` returns:

- `summary_id`
- `truth_value`
- `over`

The observed tests do not assert that `PgApi.summarize` at `reasons/pg.py:1726` returns the same keys. This is a concrete audit point: the Pg signature is `def summarize(self, summary_id, text, over, source="")`, but the return shape is not established by the observed tests.

### Specific methods worth auditing

| Method | SQLite side | Pg side | Audit concern |
|---|---|---|---|
| `show_node` | `reasons/api.py:870` | `reasons/pg.py:678` | Compare full key set and nested structures: `id`, `text`, `truth_value`, `justifications`, `dependents`, source fields, date, metadata, access filtering. |
| `summarize` | `reasons/api.py:1141` / `reasons/network.py:768` | `reasons/pg.py:1726` | Verify Pg returns `summary_id`, `truth_value`, `over`, and any expected source/metadata fields. |
| `import_json` | `reasons/api.py:2104` | `reasons/pg.py:2236` | Verify Pg returns `nodes_imported` and `nogoods_imported`, and that duplicate handling/error behavior matches. |

### Conclusion

The observed code and tests support the existence of an intended backend-equivalence contract, especially through dispatch tests and overlapping public method names. However, the observations are insufficient to prove that `PgApi` actually mirrors the SQLite branches for `show_node`, `summarize`, and `import_json`.

The missing pieces are the bodies of the API dispatch functions and `PgApi` methods, plus the portions of `tests/test_pg_dispatch.py` and `tests/test_pg.py` not included in the observations. A proper audit would compare real return values from both backends for the same operations, ideally with a parity test matrix.

## Topics to Explore

- [function] `reasons/api.py:_pg_dispatch` — Central Postgres dispatch point; inspect whether it normalizes results or returns `PgApi` output directly.
- [function] `reasons/pg.py:PgApi.import_json` — Verify the real return dict keys and duplicate/error behavior match SQLite `api.import_json`.
- [function] `reasons/pg.py:PgApi.summarize` — Compare its return shape with the SQLite `api.summarize` expectations in `tests/test_summarize.py`.
- [file] `tests/test_pg_dispatch.py` — Review the remaining lines for any backend-parity or shape assertions not present in the observed 300-line excerpt.
- [general] `backend-parity-contract-tests` — Add tests that run the same scenario against SQLite and Postgres and compare dict key sets and nested shapes.

## Beliefs

- `api-functions-route-through-pg-dispatch` — Public API functions such as `import_json` route to `PgApi` through `_pg_dispatch` when Postgres is configured, as exercised by `tests/test_pg_dispatch.py`.
- `pgapi-show-node-returns-id-truth-value-justifications` — `PgApi.show_node` is tested to return at least `id`, `truth_value`, `justifications`, and `dependents` keys in `tests/test_pg.py::TestShowNode`.
- `sqlite-import-json-returns-import-counts` — SQLite `api.import_json` is tested to return `nodes_imported` and `nogoods_imported` in `tests/test_import_json.py`.
- `pg-import-json-real-shape-is-not-tested-in-observed-dispatch-test` — The observed `tests/test_pg_dispatch.py` mocks `PgApi.import_json` and does not verify the real Postgres return shape.
- `summarize-api-shape-is-tested-only-for-sqlite-in-observations` — `tests/test_summarize.py::TestSummarizeAPI` asserts SQLite `api.summarize` returns `summary_id`, `truth_value`, and `over`, while no observed test asserts the same for `PgApi.summarize`.
