---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: verify_belief in reasons/api.py

# `verify_belief` — Explanation

## Purpose

`verify_belief` is the LLM-backed verification entrypoint for a belief in the Reason Maintenance System. It reads a node (and optionally, for derived beliefs, all of its leaf premises), fetches the source documents cited by `source`/`source_url`, and asks a model whether the belief's text is actually supported by that source. It then:

- **Records** which beliefs were `CONFIRMED` (and optionally stamps `verified_at` on them),
- **Records** which beliefs are `STALE` (and optionally retracts them), and
- Reports a per-belief verdict map.

It is the "does the source actually say this?" check that closes the loop between premise text and its cited evidence.

## Contract

**Preconditions**
- `node_id` must exist in the network (or the underlying `show_node` will raise `KeyError`).
- The node must be visible — no implicit `visible_to` filter is applied here, so `show_node` will return detail regardless of access tags (this function does **not** take a `visible_to` argument).
- `db_path` must point to a valid database when `pg_conninfo` is not given.

**Postconditions**
- On success with `dry_run=False`, the returned dict always contains `results`, `verified`, `stale`, `beliefs_checked`, `is_derived`, `dry_run`.
- If `retract=True` and any belief is marked `STALE`, `retract_node` is applied per stale belief. This can cascade OUT dependents via TMS semantics. Failed retractions are reported under `retract_failed`.
- If `pg_conninfo` is **not** set and any belief is `CONFIRMED`, `verified_at` and `updated_at` are stamped on those nodes (UTC ISO-8601 with `Z` suffix). Stamped ast is reported under `stamp_failed`.
- When `pg_conninfo` **is** set, the `verified_at` stamping is **skipped entirely** (no Postgres equivalent here).

**Invariants**
- `beliefs_checked` is the list of beliefs that were actually sent to the verifier — it equals `[target_node]` unless (`trace=True` **and** target is derived), in which case it is the deduplicated list of leaf premises.
- `is_derived` reflects whether the **target** node has justifications, not whether the checks were premises.
- `dry_run=True` short-circuits before any LLM call and before any write.

## Parameters

| Param | Type | Meaning | Edge cases |
|-------|------|---------|------------|
| `node_id` | `str` | Target belief ID | Must exist; `show_node` raises `KeyError` otherwise |
| `trace` | `bool` | If target is derived, walk to leaf premises and verify each | Ignored when target has no justifications — falls back to verifying the target itself |
| `retract` | `bool` | If a checked belief is `STALE`, call `retract_node` | Retraction of premises cascades via TMS; caught exceptions go to `retract_failed` |
| `dry_run` | `bool` | Skip LLM call + skip all writes; return only which beliefs *would* be checked | Returned dict has empty `results`/`verified`/`stale` and `dry_run: True` |
| `model` | `str` | Model name forwarded to `verify_beliefs` | Default `"claude"`; passed through opaquely |
| `timeout` | `int` | Per-call timeout (seconds) forwarded to LLM | Not enforced within this function itself |
| `db_path` | `str` | SQLite path (or ignored when `pg_conninfo` set) | Default `"reasons.db"` |
| `pg_conninfo` | object | Postgres connection info; when set, read/retract go through `_pg_dispatch`-backed APIs | Stamping of `verified_at` is skipped in this mode |
| `domain_id` | str/None | Postgres domain scoping | Passed through to backend APIs |

**Not-enforced assumptions:**
- `verify_beliefs` is assumed to return a dict keyed by belief ID whose values contain `"verdict"` and optionally `"reason"`. Any ID not in that dict is treated as `INCONCLUSIVE`.
- `Node` objects are assumed to expose `.verified_at` and `.updated_at`. Any exception during the stamping loop is swallowed and causes **all** verified IDs to land in `stamp_failed`.
- The trace logic assumes `trace_assumptions` returns the leaf premise IDs — the docstring of that function confirms this.
- `read_source` is assumed to handle a relative/`repo:path` style source string and return file contents or `None`; anything non-truthy for `source` skips the read.

## Return Value

A dict:

```
{
  "results":        dict[belief_id, verdict_dict],
  "verified":       list[str],   # IDs whose verdict == "CONFIRMED"
  "stale":          list[str],   # IDs whose verdict == "STALE"
  "beliefs_checked":list[dict],  # {"id","text","truth_value","source","source_url"}
  "is_derived":     bool,
  "dry_run":        bool,
  "retract_failed": list[str],   # only present if some retraction raised
  "stamp_failed":   list[str],   # only present if stamping raised
}
```

- On `dry_run=True`, `results` is `{}` and `verified`/`stale` are empty — callers must inspect `beliefs_checked`.
- `results` may contain unknown IDs if `verify_beliefs` returns extra entries; the loop only consumes IDs present in `beliefs_with_sources`.
- Callers must handle `KeyError` (node missing) from `show_node`/`trace_assumptions`, and must be prepared for `retract_failed`/`stamp_failed` to be set even on success (partial failure).

## Algorithm

1. **Import** `read_source` and `verify_beliefs` lazily from `.verify`, and `datetime`/`timezone`.
2. **Load the target** via `show_node(node_id, **backend)`.
3. **Determine `beliefs_to_check`:**
   - If target has justifications **and** `trace=True`:
     - `trace_assumptions` → list of leaf premise IDs.
     - Deduplicate while preserving first-seen order.
     - For each premise `pid`, `show_node(pid)` and build the belief record. On `KeyError`/`PermissionError`, record placeholders with `truth_value="UNKNOWN"` and empty source.
   - Otherwise: the single target node is the belief to check.
4. **Attach source content:** for each belief, call `read_source(source, db_path=…)` if `source` is truthy; else `None`. Result is a list of `(belief_dict, source_content)` pairs.
5. **If `dry_run`:** return immediately with empty verdicts; `is_derived` reflects target state.
6. **Call LLM:** `verify_beliefs(beliefs_with_sources, model=model, timeout=timeout)` returns a dict keyed by belief ID.
7. **Classify:**
   - `CONFIRMED` → append to `verified`.
   - `STALE` → append to `stale`; if `retract=True`, call `retract_node(bid, reason="verify: …", **backend)` in a `try`, appending to `retract_failed` on any exception.
   - `INCONCLUSIVE` (or missing/unknown verdict) → no action.
8. **Stamp confirmed beliefs** — unless `pg_conninfo` is set:
   - Open `_with_network(db_path, write=True)`.
   - For each `bid` in `verified`: if present in network, set `verified_at` and `updated_at` to `now`. If absent, append to `stamp_failed`.
   - Any exception in the `with` block → `stamp_failed = verified`.
9. **Assemble and return** the result dict; attach failure lists only when non-empty.

## Side Effects

- **Reads:** node detail, trace, source files (via `read_source`).
- **LLM call:** `verify_beliefs` (external; skipped on `dry_run`).
- **Writes:**
  - `retract_node` for each `STALE` belief when `retract=True` — this is a `write=True` `_with_network` call under the hood, and it can cascade the whole dependent graph OUT and run TMS propagation.
  - `verified_at` / `updated_at` stamping on each `CONFIRMED` belief when running in SQLite mode. This uses an internal `_with_network(write=True)` that also triggers pubsub change events (update time changes only — no truth value change, so `compute_changes` likely produces nothing for these).
- **Emits:** nothing directly, but the stamping context publishes any resulting change events via pubsub.

## Error Handling

| Source | Behavior |
|--------|----------|
| `show_node(node_id)` missing | Raises `KeyError` — **not caught**, propagates to caller |
| `show_node(premise)` missing/permission | Caught; placeholder belief recorded with `truth_value="UNKNOWN"`, empty source |
| `trace_assumptions` | Not wrapped — its `KeyError`/`PermissionError` propagate |
| `read_source` | Not wrapped — its exceptions propagate |
| `verify_beliefs` (LLM) | Not wrapped — exceptions propagate |
| `retract_node` failure | Caught (`except Exception`); ID added to `retract_failed` |
| `_with_network` stamping failure | Caught; `stamp_failed` set to entire `verified` list |
| Missing `verdict` key in LLM response | Treated as `"INCONCLUSIVE"`, silently not acted on |

The function is **not** fully defensive: the LLM layer and reads are unwrapped, so a transient LLM failure aborts the call. Only the write paths are guarded.

## Usage Patterns

- **Typical single-belief check with confirmation stamping:**
  ```python
  result = verify_belief("auth-tokens-expire-24h")
  assert "auth-tokens-expire-24h" in result["verified"]
  ```
- **Derived belief provenance check:**
  ```python
  result = verify_belief("auth-is-secure", trace=True)
  # verifies each leaf premise, not the derived claim itself
  ```
- **CI/scheduled hygiene (retract stale):**
  ```python
  result = verify_belief(node_id, retract=True, model="claude", timeout=180)
  if result.get("retract_failed"):
      log.warning("stale beliefs failed to retract: %s", result["retract_failed"])
  ```
- **Dry-run auditing:**
  ```python
  preview = verify_belief(node_id, trace=True, dry_run=True)
  print(preview["beliefs_checked"])  # no LLM call, no stamps
  ```

**Caller obligations**
- Check `retract_failed`/`stamp_failed` — success signal in `verified`/`stale` does **not** guarantee all writes succeeded.
- In Postgres mode, do not rely on `verified_at` being updated — it is silently skipped.
- `trace=True` is a no-op if the target has no justifications; callers wanting to gossip-verify should check `is_derived` on the result or the node itself first.

## Dependencies

- **`.verify`** — `read_source`, `verify_beliefs` (LLM adapter and source reader)
- **`.api`** (same module): `show_node`, `trace_assumptions`, `retract_node`, `_with_network`
- **`.storage` / `.network`** — via `_with_network` for the write path
- **stdlib:** `datetime`, `timezone`

## Topics to Explore

- [file] `reasons/verify.py` — Implements `read_source` and `verify_beliefs`; the truth of the whole pipeline depends on how it classifies and what verdict keys it returns
- [function] `reasons/api.py:trace_assumptions` — Determines which leaf premises get verified when `trace=True`; its filtering rules directly affect `beliefs_checked`
- [function] `reasons/api.py:retract_node` — The cascade mechanism invoked when `retract=True`; a single retract can ripple OUT many dependents
- [general] `verified_at-lifecycle` — How `verified_at` is written, read, and used across `show_node`, `check_stale`, and export — the interaction between this stamp and `pinned_sha` matters for staleness
- [general] `postgres-parity` — Which APIs have real `PgApi` implementations vs. inferring behavior from `pg_conninfo` branching; the skipped stamping path is one of several asymmetries

## Beliefs

- `verify-belief-requires-verify-module` — `verify_belief` depends on `.verify`'s `read_source` and `verify_beliefs` and will raise at call time if either is missing.
- `trace-requires-justifications` — `trace=True` has no effect unless the target node has at least one justification; otherwise the target itself is verified.
- `dry-run-skips-llm-and-writes` — With `dry_run=True`, the function returns before `verify_beliefs` is called and makes no database writes.
- `postgres-mode-skips-verified-at-stamping` — When `pg_conninfo` is set, `verified_at`/`updated_at` stamping is skipped entirely even for CONFIRMED beliefs.
- `retract-failures-are-swallowed` — `retract_node` exceptions during stale handling are caught and reported in `retract_failed` rather than propagated.
- `verified-at-is-stamped-not-truth-affecting` — Stamping `verified_at` in SQLite mode goes through `_with_network(write=True)` but does not change `truth_value`, so it triggers no retraction cascade.
