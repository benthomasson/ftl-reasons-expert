---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: report_belief in reasons/api.py

## Overview

`report_belief` is a **read-only orchestration function** in `reasons/api.py`. Its job is to assemble everything needed to produce a human/LLM-readable "evidence report" for a single belief, then delegate the narrative generation to `reasons.report_belief.report_belief`. It does **not** compute the report itself — it gathers the three inputs (node detail, explanation steps, traced premises), optionally enriches those premises with source-document chunks, and hands them off.

## Purpose

A belief in the RMS is derived from root premises (assumptions/axioms). To justify a belief to a human reviewer, you need:

1. The belief's own record (text, truth value, justifications, metadata).
2. An explanation of *why* it is currently `IN`/`OUT` (from `explain_node`).
3. The set of premises it transitively rests on (from `trace_assumptions`).
4. Optionally, the original source material those premises cite (from a separate source-chunk store).

`report_belief` exists to combine those into a single call that the CLI / LangGraph tools can invoke without duplicating orchestration and error-swallowing logic.

## Contract

- **Precondition:** `node_id` must exist in the network at `db_path` (or the PG domain behind `pg_conninfo`/`domain_id`). If not, `show_node(node_id)` raises `KeyError`.
- **Postcondition:** returns `{"report": str, "premise_count": int}`. `premise_count` equals the number of premise IDs returned by `trace_assumptions`, *including* premises that failed to resolve (they are still counted, just with placeholder fields).
- **Invariants:**
  - The network is **never mutated**. All work is read-only against the RMS store.
  - `premise_count == len(premises_data)`, and `premises_data` has exactly one entry per ID in `trace_data["premises"]` (no dedup, no filtering).
  - `source_chunks` is either empty (when `sources_db` is `None`) or a dict keyed by premise ID with list values (possibly empty lists on error).

## Parameters

| Parameter | Type | Meaning |
|---|---|---|
| `node_id` | `str` | The belief to report on. Must exist. |
| `sources_db` | `str \| None` | Optional path to a *separate* source-chunk database (used by `reasons.ask.search_source_chunks`). When `None`, source enrichment is skipped entirely. |
| `model` | `str` | LLM model name forwarded verbatim to the report helper. Default `""` — the helper's own default is used. |
| `timeout` | `int` | LLM timeout in seconds, forwarded to the helper. Default `300`. |
| `db_path` | `str` | Path to the RMS SQLite DB. Defaults to `DEFAULT_DB` (`"reasons.db"`). |
| `pg_conninfo` / `domain_id` | opaque | If set, the **sub-calls** (`show_node`, `explain_node`, `trace_assumptions`) dispatch to the PostgreSQL backend. The function itself does not have a `_pg_dispatch` branch. |

**Edge cases:**
- `sources_db` pointing at a non-existent/incompatible DB: the per-premise `search_source_chunks` call raises, the exception is swallowed, and that premise gets an empty list.
- A premise ID in `trace_data["premises"]` that is missing from the network (e.g., dangling reference) yields a placeholder entry instead of failing the whole call.

## Return Value

```python
{
    "report": "<string produced by _report_belief>",
    "premise_count": len(premises_data),
}
```

The caller must handle:
- `KeyError` if `node_id` does not exist.
- Whatever `_report_belief` raises (e.g., LLM/network errors, timeout errors) — those are **not** caught here.
- The `report` string is opaque; it is not validated or parsed.

## Algorithm (step by step)

1. **Lazy import** `_report_belief` from `reasons.report_belief` (avoids import cycle / heavy import at module load).
2. Build a `backend` dict `{db_path, pg_conninfo, domain_id}` for reuse across sub-calls.
3. `node_detail = show_node(node_id, **backend)` — full node record. This is the existence check (raises `KeyError` on unknown node).
4. `explain_data = explain_node(node_id, **backend)` — returns `{"steps": [...]}` explaining the current truth value.
5. `trace_data = trace_assumptions(node_id, **backend)` — returns `{"node_id": ..., "premises": [...]}`.
6. **Premise enrichment loop:** for each `pid` in `trace_data["premises"]`:
   - Call `show_node(pid, **backend)`.
   - On success, record `{id, text, truth_value, source}`.
   - On `KeyError` or `PermissionError`, record `{id, text:"", truth_value:"UNKNOWN", source:""}` — **the failure is swallowed and the premise is still counted**.
7. **Source-chunk lookup (optional):** if `sources_db` is truthy:
   - Import `search_source_chunks` from `reasons.ask`.
   - For each premise with non-empty `text`, call `search_source_chunks(text, sources_db, top_k=3)`.
   - On any exception, store `[]` for that premise ID (again swallowed).
8. **Delegate:** call `_report_belief(node_id, node_detail, explain_data["steps"], premises_data, source_chunks, model=model, timeout=timeout)`.
9. **Wrap:** return `{"report": <helper result>, "premise_count": len(premises_data)}`.

## Side Effects

- **No writes to the RMS network.** All RMS access is via read-only functions (`show_node`, `explain_node`, `trace_assumptions`), and none of those open a `write=True` network context.
- **Additional reads** against `db_path` (via `show_node` which itself also reads `node_tags` and `node_sources` tables using a fresh `sqlite3.connect`).
- **Optional reads** against `sources_db`.
- **External I/O:** the delegated `_report_belief` is expected to call an LLM (per `model`/`timeout`), which is network I/O. That call happens synchronously in this function.
- No caching, no pubsub events.

## Error Handling

Errors are handled at **three distinct levels**:

1. **Fatal (propagates):**
   - `KeyError` from `show_node(node_id)` if the target belief is missing.
   - Any exception from `_report_belief` (LLM failure, timeout, schema mismatch in report module).
   - Any exception from the top-level `show_node`/`explain_node`/`trace_assumptions` other than the types caught below.

2. **Per-premise swallow:**
   - `show_node(pid)` failures of type `KeyError` or `PermissionError` → placeholder replaces the premise. `PermissionError` is *defensive* here because `visible_to` is never passed, so the visibility gate inside `show_node` never triggers.
   - Other exceptions from `show_node(pid)` would propagate — the code only catches those two.

3. **Per-premise source lookup swallow:**
   - `search_source_chunks(...)` — *any* exception is caught (`except Exception`), so a broken/missing `sources_db` degrades silently to `{pid: []}`.

**Implicit assumption:** the caller cannot distinguish "premise text was empty" from "premise failed to load" from "source lookup failed" in the return value — the `report` string is the only place that might surface this.

## Usage Patterns

Typical call chain:

```
CLI `reasons report-belief <node_id> [--sources-db path]`
  └─> api.report_belief(node_id, sources_db=..., model=..., db_path=...)
        └─> report_belief.report_belief(...)   # LLM narrative
```

Caller obligations:
- Ensure the network is initialized (`init_db`) and `node_id` is present *before* calling; otherwise `KeyError`.
- If `sources_db` is provided, ensure it is the kind of chunk store `reasons.ask.search_source_chunks` expects — otherwise you get silent empty lists, not an error.
- Handle LLM failures from the underlying report module; `report_belief` itself does not retry or timeout-guard beyond forwarding `timeout`.
- **Do not rely on this function for access control** — it never accepts or forwards `visible_to`, so any node the process can read is reported.

## Dependencies

- **Same module:** `show_node`, `explain_node`, `trace_assumptions`, `DEFAULT_DB`.
- **Lazy imports:** `.report_belief.report_belief` (the narrative generator), `.ask.search_source_chunks` (only when `sources_db` is set).
- **Transitive:** `.storage`, `.network`, and `.pg` are reached via the sub-call functions; PG dispatch happens inside each sub-call, not here.

## Notes / gotchas

- **Not a PG-dispatched function.** Unlike `challenge`, `supersede`, `report_gated`, etc., this function has no `if pg_conninfo: return _pg_dispatch(...)` branch. It supports PG only indirectly, by passing `pg_conninfo`/`domain_id` into each sub-call. The narrative generation and source lookup run locally regardless of backend.
- **`premise_count` is "premises traced", not "premises successfully resolved".** If a premise is dangling or inaccessible, it is still included in the count with `truth_value: "UNKNOWN"`.
- **Visibility is not applied.** The signature deliberately drops `visible_to` (compare `explain_node`, `show_node`, `trace_assumptions`, which all accept it), so this function bypasses access-tag filtering on both the target belief and its premises.
- **`trace_data["premises"]` and `explain_data["steps"]` are key-dependent.** The type system does not enforce these shapes; the function assumes both sub-call return dicts have these keys.
- **The `PermissionError` branch on premise `show_node` is currently dead code** given `visible_to=None`, but is kept presumably in anticipation of a future version accepting visibility filters.

## Topics to Explore

- [file] `reasons/report_belief.py` — The narrative generator that turns `node_detail`, explanation steps, premises, and source chunks into an LLM prompt and markdown report; its signature is the real contract this function must satisfy.
- [function] `reasons/api.py:trace_assumptions` — Defines what "root premises" means for the report (walking justification antecedents backward, filtering by visibility when provided).
- [function] `reasons/ask.py:search_source_chunks` — Source-chunk retrieval behind `sources_db`; understanding its schema and failure modes clarifies why the lookup is wrapped in a blanket `except Exception`.
- [file] `tests/test_report_belief.py` — Shows the expected behaviour for happy-path and for missing/dangling premises.
- [general] `postgres-backend-dispatch-pattern` — How most `reasons/api.py` functions dispatch to `PgApi` via `_pg_dispatch`, and why `report_belief` deviates by routing PG through sub-calls only.

## Beliefs

- `report-belief-delegates-narrative-generation` — `report_belief` gathers data and then delegates the actual report string to `reasons.report_belief.report_belief`; it never formats output itself.
- `report-belief-is-read-only` — The function opens the RMS store only through read-only sub-calls (`show_node`, `explain_node`, `trace_assumptions`) and performs no writes.
- `source-chunk-lookup-is-optional` — Source chunk enrichment from `sources_db` runs only when the parameter is truthy; otherwise `source_chunks` is an empty dict passed to the report helper.
- `premise-load-failures-are-swallowed` — Premises whose `show_node` raises `KeyError` or `PermissionError` are recorded as `{"truth_value": "UNKNOWN", ...}` instead of aborting the report.
- `report-belief-does-not-apply-visibility-filtering` — The signature omits `visible_to` and never forwards it to sub-calls, so access-tag filtering is not applied to either the target belief or its premises.
- `report-belief-has-no-direct-pg-dispatch` — Unlike most write-oriented functions in this module, `report_belief` has no `_pg_dispatch` branch; PostgreSQL support flows only through `pg_conninfo`/`domain_id` passed to its sub-calls.
