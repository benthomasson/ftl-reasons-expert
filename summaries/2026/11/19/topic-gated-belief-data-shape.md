---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Trace how `list-gated` and `show` data are shaped upstream so the required keys and counts are correct.

## Trace

The observations show the **consumers** and **test contracts** for `list-gated` and `show`, but not the full producer bodies. Still, the required upstream shape can be reconstructed from `reasons/report_gated.py`, `reasons/api.py`, and the tests.

### 1. `list-gated` upstream path

`list-gated` starts at `reasons/api.py:3628`:

```python
def list_gated(
```

When the Postgres backend is selected, it dispatches at `reasons/api.py:3639`:

```python
return _pg_dispatch(pg_conninfo, domain_id, "list_gated", ...)
```

That means the Postgres implementation must match the same contract at `reasons/pg.py:1258`:

```python
def list_gated(self, visible_to=None):
```

The CLI entry point is `reasons/cli.py:2114 cmd_list_gated`, which calls `api.list_gated(...)` at `reasons/cli.py:2116`. The parser is registered at `reasons/cli.py:3648` and mapped to the command at `reasons/cli.py:3996`.

The MCP wrapper is `reasons/mcp_server.py:557 list_gated`, which calls `api.list_gated(db_path=_get_db())` at line 565. Tests at `tests/test_mcp_server.py:251` parse the result as JSON, and `tests/test_mcp_server.py:633` exercise `output_format="names-only"`.

The report path is important: `reasons/api.py:3708` calls:

```python
gated_data = list_gated(**vis, **backend)
```

and passes that object into `reasons/report_gated.py`.

### 2. Required shape of `gated_data`

`reasons/report_gated.py:_build_structured_report` and `_format_data_for_prompt` require `gated_data` to contain:

```python
{
    "blocker_count": int,
    "gated_count": int,
    "blockers": {
        "<blocker-id>": {
            "text": "<blocker text>",
            "gated": [
                {"id": "<gated-node-id>", "text": "<gated node text>"},
                ...
            ],
        },
        ...
    },
}
```

The important detail is that `report_gated` does **not** recompute the top-level counts. It reads them directly:

```python
blocker_count = gated_data["blocker_count"]
gated_count = gated_data["gated_count"]
```

So upstream `list_gated` must guarantee:

- `blocker_count == len(gated_data["blockers"])`
- `gated_count == sum(len(b["gated"]) for b in gated_data["blockers"].values())`

The test helper in `tests/test_report_gated.py:_make_gated_data` encodes exactly that relationship:

```python
gated_count = sum(len(b["gated"]) for b in blockers.values())
return {
    "blockers": blockers,
    "blocker_count": len(blockers),
    "gated_count": gated_count,
}
```

`tests/test_report_gated.py:TestReportGatedApi.test_counts_match` then verifies the live API returns matching counts for a fixture with one blocker and one gated belief:

```python
assert result["blocker_count"] == 1
assert result["gated_count"] == 1
assert result["retracted_count"] == 1
```

The report module also iterates blockers and gated entries directly:

- `blockers.items()` expects blocker IDs as keys
- `bdata["text"]` is printed as the blocker description
- `bdata["gated"]` is used for `len(gated)` and for each `g["id"]` / `g["text"]`

So if `list_gated` omits `text`, `gated`, `id`, or `text` on a gated entry, `report_gated` will fail.

### 3. `show` upstream path and impact counts

The `show` command is registered at `reasons/cli.py:3186`:

```python
p = sub.add_parser("show", help="Show node details")
p.add_argument("node_id", help="Node to show")
p.add_argument("--visible-to", ...)
```

The API entry point is `reasons/api.py:870`:

```python
def show_node(node_id: str, visible_to: list[str] | None = None, db_path: str = DEFAULT_DB, ...)
```

The observations do not include `cmd_show` or the body of `show_node`, so the exact returned dict keys are not visible. However, `tests/test_cli.py:TestShow` defines the user-facing contract:

- Premise show output contains `ID:     a`
- Text output contains `Text:   Premise A`
- Status output contains `Status: IN`
- Premise output contains `Premise (no justifications)`
- Derived output contains `SL(a)`
- Dependent output contains `Dependents: b`
- Retraction output contains `Retract reason: Fixed in PR #1`
- Access filtering produces `Access denied` when `--visible-to` excludes the node

For `report_gated`, the impact-count shape explicitly requires `dependent_count`:

- `retracted_premises` is a list of objects with `id`, `text`, `retract_reason`, and `dependent_count`
- `blocker_details` is a mapping from blocker ID to an object with `dependent_count`

The report module uses these like this:

```python
for rp in sorted(retracted_premises, key=lambda r: -r["dependent_count"]):
    ...
    text = rp["text"][:80]
    reason = rp["retract_reason"]
```

and:

```python
detail = blocker_details.get(bid, {})
dep_count = detail.get("dependent_count", 0)
```

So `show` / dependent-count data must be shaped upstream into scalar `dependent_count` values before `report_gated` can produce correct impact numbers.

### 4. What is missing from these observations

To fully trace the exact SQL and field mapping, the observations would need:

- The body of `reasons/api.py:list_gated` after line 3628, especially the non-Postgres path
- The body of `reasons/pg.py:1258 PgApi.list_gated`
- The body of `reasons/api.py:870 show_node`
- The `cmd_show` implementation in `reasons/cli.py` near the parser at line 3186
- The `api.report_gated` implementation around `reasons/api.py:3708`, which must build `retracted_premises`, `blocker_details`, and `retracted_count`
- The assertions in `tests/test_api.py:412-459` and `tests/test_pg.py:378-413`

Without those, the exact producer logic is not visible. The observed contract, however, is clear: `list_gated` must return precomputed blocker/gated counts plus a nested blocker-to-gated-node mapping, and `show`-derived data must provide dependent counts for impact reporting.

## Topics to Explore

- [function] `reasons/api.py:list_gated` — Confirm how `blockers`, `blocker_count`, and `gated_count` are computed and whether the SQLite path and Postgres path share the same shape.
- [function] `reasons/pg.py:PgApi.list_gated` — Verify Postgres parity with the SQLite `list_gated` contract, especially ordering and count semantics.
- [function] `reasons/api.py:show_node` — Inspect the exact returned keys for status, justifications, dependents, and retract reason.
- [function] `reasons/cli.py:cmd_show` — See how `show_node` data is formatted and whether it exposes `dependent_count` or only a dependents list.
- [function] `reasons/api.py:report_gated` — Trace how `retracted_premises`, `blocker_details`, and `retracted_count` are assembled before calling `reasons/report_gated.py`.

## Beliefs

- `report-gated-reads-precomputed-counts` — `reasons/report_gated.py` uses `gated_data["blocker_count"]` and `gated_data["gated_count"]` directly instead of deriving them from `blockers`.
- `list-gated-postgres-dispatch-requires-parity` — `reasons/api.py:list_gated` delegates to `PgApi.list_gated` at `reasons/pg.py:1258` when the Postgres backend is selected.
- `blocker-entries-require-text-and-gated-list` — Each entry in `gated_data["blockers"]` is consumed as an object containing `text` and a `gated` list of objects with `id` and `text`.
- `retracted-premises-require-dependent-count` — `report_gated` sorts retracted premises by `dependent_count` and formats each using `id`, `text`, and `retract_reason`.
- `show-tests-define-cli-output-contract` — `tests/test_cli.py:TestShow` asserts `show` output includes `Dependents: b` and `Retract reason: Fixed in PR #1`.
