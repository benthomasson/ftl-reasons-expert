---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: report_gated in reasons/api.py

# `report_gated` — `reasons/api.py`

## Purpose

`report_gated` is a top-level, JSON-serializable **read-only API entry point** that aggregates the RMS' "state of trouble" into a single report. It gathers four kinds of signals:

1. Counts of IN and OUT beliefs (the overall shape of the belief state).
2. A list of **retracted premises** (OUT premises whose metadata carries `retract_reason`).
3. The set of **active gates** (`OUT` nodes blocked by `IN` outlist nodes — the "gated" signal produced by `list_gated`).
4. Per-blocker dependent counts to gauge blast radius.

It then delegates the human-readable rendering (typically LLM-backed) to `reasons.report_gated.report_gated` and returns the report string plus summary counts. It exists so CLI/tool wrappers can produce a "what's broken right now?" narrative without reimplementing the data collection.

## Contract

- **Preconditions**
  - `db_path` points at a valid RMS SQLite file — or is a non-existent path, in which case `_with_network` will *create* an empty database as a side effect (via `Storage(db_path)`).
  - If `pg_conninfo` is set, the backend must support `list_nodes`, `show_node`, and `list_gated` (they do — see below).
- **Postconditions**
  - Returns a 4-key dict; no database writes are performed by this function.
  - Under SQLite, the network is loaded once (via the `_network_cache` keyed by path+mtime) and reused across the sub-calls, since none of them write.
- **Invariants / implicit assumptions**
  - `gated_data` **must** contain `"blockers"`, `"blocker_count"`, `"gated_count"` (guaranteed by `list_gated`'s contract).
  - Only **premises** (nodes with no justifications) contribute to `retracted`. A derived node can have `retract_reason` metadata but still be excluded.
  - Only nodes whose `metadata["retract_reason"]` is present are counted as retracted — the code never checks `_retracted` or a `retracted_at` timestamp.
  - If `pg_conninfo` is set, each helper dispatches to the PG backend, but `_report_gated` (the submodule) is always invoked **locally** — no PG dispatch for report rendering.

## Parameters

| Param | Type | Default | Notes |
|---|---|---|---|
| `visible_to` | `list[str] \| None` | `None` | Access-tag subset filter. `None` means "no filtering." Threaded through every helper as `**vis`. |
| `model` | `str` | `""` | LLM model identifier passed straight to `_report_gated`. Empty string means "use module default." |
| `timeout` | `int` | `300` | Seconds; passed to `_report_gated` (LLM call budget). |
| `db_path` | `str` | `DEFAULT_DB` (`"reasons.db"`) | SQLite path — ignored if `pg_conninfo` is set. |
| `pg_conninfo` | anything | `None` | If truthy, routes `list_nodes`/`show_node`/`list_gated` to `PgApi`. |
| `domain_id` | anything | `None` | PG domain identifier, paired with `pg_conninfo`. |

## Return Value

```python
{
    "report":        str,  # rendered narrative from reasons.report_gated.report_gated
    "blocker_count": int,  # number of distinct IN outlist nodes gating OUT beliefs
    "gated_count":   int,  # total OUT beliefs currently gated
    "retracted_count": int # len(retracted) — OUT premises with retract_reason
}
```

Callers should not assume `report` is plain text — it is whatever `_report_gated` returns (currently a string, per the docstring). `blocker_count`/`gated_count` come directly from `list_gated`; `retracted_count` is `len(retracted)` as built locally.

## Algorithm (step by step)

1. **Set up parameter bundles.**
   - `backend = dict(db_path=db_path, pg_conninfo=pg_conninfo, domain_id=domain_id)`
   - `vis = dict(visible_to=visible_to)`
   These are splatted into every helper call via `**vis, **backend`.

2. **Count nodes.**
   - `list_nodes(status="IN", **vis, **backend)` → only `["count"]` used.
   - `list_nodes(status="OUT", **vis, **backend)` → only `["count"]` used.

3. **Collect retracted premises.**
   - `list_nodes(status="OUT", premises_only=True, **vis, **backend)` returns the OUT *premises*.
   - For each, call `show_node(node["id"], **vis, **backend)` to fetch metadata.
   - Keep only those with `metadata["retract_reason"]`; build `{id, text, retract_reason, dependent_count}`.
   - Note: `dependent_count` here comes from `list_nodes` output (`node["dependent_count"]`), not from `show_node`.

4. **Collect gated beliefs and blockers.**
   - `list_gated(**vis, **backend)` returns `{"blockers": {...}, "gated_count", "blocker_count"}`.

5. **Enrich blockers.**
   - For each blocker ID, try `show_node(bid, **vis, **backend)` to count `dependents`.
   - Catch `KeyError` and `PermissionError` and default `detail = {}`. Consequence: an **invisible blocker** (visible filter applies to gated node, not to blocker in `list_gated`) yields `dependent_count = 0`.

6. **Delegate rendering.**
   - `report = _report_gated(gated_data, retracted, blocker_details, in_count, out_count, model=model, timeout=timeout)`
   - `_report_gated` is `reasons.report_gated.report_gated`, imported inside the function body.

7. **Assemble and return** the 4-key dict.

The interesting bit is step 5: `list_gated` filters gated (OUT) nodes by `visible_to`, not blockers. So a blocker that is invisible to the caller still appears as a blocker key, but `show_node` will raise `PermissionError`, and the try/except quietly reduces its `dependent_count` to `0`. This is by design (report the gating relationship even if the blocker isn't visible) but it means the "impact" number is not trustworthy for hidden blockers.

## Side Effects

- **Read-only** relative to the network. `list_nodes(status=...), list_nodes(premises_only=...), show_node, list_gated` are all read paths.
- **Filesystem.** Under SQLite, `_with_network` will **create** `db_path` if it doesn't exist (via `Storage(db_path)`), because the cache check falls into the `else` branch when `os.path.getmtime` raises. So a call against a typo'd path silently creates an empty database.
- **In-memory cache.** All sub-calls share the `_network_cache` entry for `db_path` (keyed by `(db_path, mtime)`), so the disk is read at most once per call, but the cache is populated as a side effect.
- **External I/O.** `_report_gated` is very likely to call an LLM (it takes `model` and `timeout`); failures there (network, auth, timeouts) are not caught by this function.
- **PG mode.** One round-trip per `list_nodes`/`show_node` call — potentially `2 + |out_premises| + |blockers|` PG queries.

## Error Handling

| Location | Behavior |
|---|---|
| `show_node(node["id"])` in the retracted loop | **Not wrapped.** `KeyError`/`PermissionError` will propagate. In practice the nodes came from `list_nodes(premises_only=True, visible_to=...)`, so the visibility guarantee usually holds — but the visibility filter is applied to the *node*, so this is safe in the normal path. |
| `show_node(bid)` in the blocker loop | Wrapped: `except (KeyError, PermissionError)`. Degrades to `detail = {}` → `dependent_count = 0`. This is *required* because blockers are **not** filtered by `visible_to` in `list_gated`. |
| `_report_gated(...)` | No wrapping. LLM/rendering failures (network, parse errors, timeouts) propagate to the caller. |
| PG dispatch | If the PG backend doesn't implement one of the required methods, `AttributeError` from `getattr(pg, method_name)` propagates. |

Returned values never contain an `"error"` key — this function either returns a fully-formed report dict or raises.

## Usage Patterns

Typical callers:

```python
# Simple
result = report_gated()
print(result["report"])

# Scoped by access
result = report_gated(visible_to=["public", "reviewers"], model="claude")

# PG-backed
result = report_gated(pg_conninfo="postgres://...", domain_id="team-a")
```

- Callers should call this **after** a fresh propagate/run-cycle if they expect to see current OUT-cascade state.
- Callers must not assume `report` is empty when `gated_count == 0` — the report may still describe retractions or general state.
- The **`premises_only=True` filter** on retracted nodes is a real behavioral constraint: retracted *derived* nodes (with justifications) are silently absent from `retracted_count`, even if their metadata has `retract_reason`. Anyone relying on "all retractions" should look elsewhere (e.g. `list_nodes(status="OUT", ...)` directly).
- Under the hood, `retract_node()` does **not** appear to write `metadata["retract_reason"]` itself — it just calls `net.retract(node_id, reason=...)`. Only code paths that explicitly set metadata (e.g. `mark_duplicate`, `mark_superseded`, potentially `Network.retract`) will cause nodes to appear in the retracted section. This is an assumption baked into the report: *the retracted list is not a complete audit of retractions; it is a view over metadata that some retraction paths set and others may not.*

## Dependencies

- `list_nodes` — counts and OUT-premise enumeration (`reasons/api.py`).
- `show_node` — metadata + dependent counts (`reasons/api.py`).
- `list_gated` — gate detection (`reasons/api.py`).
- `_report_gated` — actual narrative rendering (`reasons/report_gated.py`).
- `_with_network` — load / cache / read-only lifecycle (`reasons/api.py`).
- `_pg_dispatch` — PG backend shim (invoked *inside* the helpers, not here).
- Indirect: `Network`, `Storage`, `Justification`, `pubsub` (for the broader module), and whatever LLM client `reasons.report_gated` uses.

## Topics to Explore

- [file] `reasons/report_gated.py` — Where the LLM prompt template, batching, and output formatting for the report live; the pure-data shape this function passes in.
- [function] `reasons/api.py:list_gated` — The upstream gate detector; explains why blockers are not visibility-filtered and how `blocker_count`/`gated_count` are computed.
- [function] `reasons/api.py:retract_node` — The canonical retraction entry point; check whether it writes `metadata["retract_reason"]` before calling `net.retract`, which determines whether the retracted section is complete.
- [function] `reasons/api.py:list_nodes` — The single helper doing the top-level visibility filter and `premises_only` cut; many semantic questions about this report resolve there.
- [general] `network-cache-mtime-invalidation` — How `_network_cache` interacts with concurrent writers and why read-only report APIs are effectively a single load per call.

## Beliefs

- `report-gated-sums-counts-from-list-gated` — `blocker_count` and `gated_count` in the returned dict are copied directly from `list_gated`'s output, not recomputed.
- `report-gated-uses-premises-only-for-retracted` — Retracted entries are confined to `list_nodes(status="OUT", premises_only=True, ...)`, so retracted nodes that still have justifications are excluded.
- `report-gated-swallows-blocker-lookup-errors` — `show_node` failures for blockers are caught as `(KeyError, PermissionError)` and reduce `dependent_count` to `0`.
- `report-gated-does-not-catch-retracted-lookup-errors` — `show_node` calls inside the retracted-node loop are unwrapped; `KeyError`/`PermissionError` propagate to the caller.
- `report-gated-delegates-render-to-report-gated-module` — The narrative string is produced by `reasons.report_gated.report_gated`, which is always invoked locally even when `pg_conninfo` is set.
