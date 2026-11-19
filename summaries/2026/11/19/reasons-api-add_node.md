---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Function: add_node in reasons/api.py

# `add_node` — Explanation

## Purpose

`add_node` is the primary public entry point for introducing a belief into the Reason Maintenance System. It creates a new node, attaches zero or more justifications (SL/CP/outlist), optionally scopes it to a namespace (by adding a dependency on `{namespace}:active`), derives metadata from the caller's provenance hints, then persists everything by going through `_with_network(..., write=True)`. The truth value of the new node is recomputed as part of the TMS propagation and returned to the caller.

It exists so that CLI, LangGraph tools, and scripts can add beliefs without managing `Storage`/`Network` lifecycle themselves — the function opens, mutates, saves, and closes the DB in one atomic-looking operation.

## Contract

**Preconditions**
- If `source_type` is non-empty, it must be a member of `SOURCE_TYPES = {"code", "document", "self-description", "derived"}`; otherwise `ValueError`.
- `sl`, `cp`, and `unless` are comma-separated ID strings; commas are not escapable, so any node ID containing a comma breaks parsing.
- `node_id` is not checked for prior existence here — it delegates to `Network.add_node` (uniqueness/overwrite semantics are that level's responsibility).
- Referenced antecedent IDs are **not** validated to exist; dangling references pass through to `Network.add_node`.

**Postconditions (SQLite path)**
- The node exists in the network after `_with_network` exits, with `truth_value` recomputed by the TMS.
- If `namespace` is set, `{namespace}:active` exists as a premise with `metadata={"agent": namespace, "role": "agent_premise"}`.
- `{namespace}:active` is prepended as an antecedent to `justifications[0]` **only** — the other justifications in `any_mode` do not receive this dependency (see "Assumptions" below).
- Metadata keys populated: `access_tags` (sorted, deduped), `example` (if not `None`), `source_type` (if non-empty), `accepted_pr` (if non-empty). `source`/`source_url` are passed as first-class node fields, not metadata.

**Invariants**
- At most one of `sl` / `cp` / "unless-only" produces justifications; the priority is `sl` > `cp` > `unless`.
- An outlist-only call (`unless` set, `sl` and `cp` empty) produces an SL justification with empty antecedents — i.e. a premise that holds unless something in the outlist is believed.
- If `namespace` is set and no justification was produced, a synthetic SL justification depending on `{namespace}:active` is added, so **all namespace-scoped nodes become derived, never premises**.

## Parameters

| Param | Meaning | Edge cases |
|---|---|---|
| `node_id` | Node identifier | If `namespace` is set and `node_id` already contains `:`, no prefix is added (allows cross-namespace references). |
| `text` | Human-readable belief text | Passed through unchanged. |
| `sl` | Comma-separated SL antecedents | Priority over `cp`. Parsed with `.strip()` on each segment. |
| `cp` | Comma-separated CP antecedents | Only used when `sl` is empty. |
| `unless` | Comma-separated outlist IDs | Can stand alone (outlist-only "gated premise"). |
| `label` | Justification label | Propagated to the (possibly synthetic) justification. |
| `source` / `source_url` | Provenance | Assigned to node fields, not metadata. |
| `namespace` | Optional prefix | Auto-creates `{namespace}:active`. Prefix is **not** applied if ID already contains `:`. |
| `any_mode` | OR-expansion of SL | Only applies when `sl` has >1 antecedent. Ignored for `cp`. Raises `NotImplementedError` on PG. |
| `access_tags` | Data-providing tags | Sorted and deduped into `metadata["access_tags"]`. |
| `example` | Example text | Stored verbatim when `not None` (distinguishes `""` from absent). |
| `source_type` | Enumerated source kind | Must be in `SOURCE_TYPES`; error otherwise. |
| `accepted_pr` | PR reference | Stored in metadata if non-empty. |
| `db_path` | SQLite path | Ignored when `pg_conninfo` is set. |
| `pg_conninfo` / `domain_id` | PG routing | When set, entire call dispatches to `PgApi.add_node`. |

## Return Value

`{"node_id": str, "truth_value": "IN"|"OUT", "type": str, "premise_count": int}`

- `node_id` reflects the **resolved** ID (i.e. the namespaced form if a namespace was applied).
- `type` is `justifications[0].type` if any justifications exist, else `"premise"`.
- `premise_count` is `max(len(j.antecedents) for j in justifications)`, defaulting to `0`. Note this is the *widest* justification, not the count of justifications.
- On the PG path the return value is whatever `PgApi.add_node` returns (shape not enforced here).

## Algorithm

1. **PG short-circuit**: if `pg_conninfo`, `any_mode` raises `NotImplementedError`; otherwise dispatch to `_pg_dispatch(..., "add_node", ...)`. Notice `source_type` and `accepted_pr` are **not forwarded** on this path.
2. **Parse `unless`** into `outlist` (strip + drop empties).
3. **Build justifications** in priority order:
   - `sl` present → split; if `any_mode` and >1 antecedent, create one SL justification per antecedent (OR semantics); otherwise one SL with the full antecedent list.
   - `cp` present → one CP justification.
   - `unless` present with no `sl`/`cp` → SL justification with empty antecedents (pure outlist gate).
4. **Open the network with write intent** via `_with_network(db_path, write=True)`.
5. **Namespace block**:
   - Resolve `node_id` (prefix unless already contains `:`).
   - Ensure `{namespace}:active` exists (adds it as a premise with `role="agent_premise"`).
   - If a justification exists: prepend `active_id` to `justifications[0].antecedents` **only**.
   - If none exists: create synthetic SL with `antecedents=[active_id]` and label `label or f"added by agent: {namespace}"`.
   - Re-resolve namespace on every justification's antecedents and outlist.
6. **Validate `source_type`** against `SOURCE_TYPES`.
7. **Build metadata** dict from `access_tags` (sorted set), `example`, `source_type`, `accepted_pr`.
8. **Delegate** to `net.add_node(...)` (triggers TMS recompute and adds to dependents index).
9. **Compute** `jtype` and `max_premises`, build and return the result dict. The device exits the `_Ctx` (saves, publishes pubsub, updates cache).

## Side Effects

- **DB write**: `_with_network(write=True).__exit__` calls `self.store.save(self.network)` on successful exit.
- **Namespace premise creation**: may add one node (`{namespace}:active`) as a side-effect.
- **Pubsub publishing**: if subscribers exist, `_with_network.__exit__` computes truth-value diffs before/after and publishes change events.
- **Cache mutation**: `_network_cache[db_path]` is updated with the new mtime after save.
- **PG side effects** are whatever `PgApi.add_node` performs; the SQLite path is bypassed entirely.

## Error Handling

- `ValueError` — invalid `source_type`.
- `NotImplementedError` — PG path with `any_mode=True`.
- `FileExistsError`, `KeyError`, database errors — propagated from underlying `Storage` / `Network` / `PgApi` calls; the function does not swallow them.
- Errors **inside** the `with` block propagate out; `_with_network.__exit__` still closes the storage but skips `save()` when an exception was raised. So an exception mid-add leaves the DB unchanged (no partial write).
- No validation of `access_tags` contents (they are opaque strings from the metadata layer's perspective), and no validation that referenced antecedents exist.

## Assumptions Not Enforced by the Type System

- **`any_mode` + `namespace` is inconsistent**: only `justifications[0]` receives `{ns}:active` as an antecedent. In OR-mode with N>1 antecedents, N−1 justifications do not depend on the namespace premise, so retracting `{ns}:active` would not cascade all of them OFF. This looks like a latent bug.
- Comma-separated inputs assume node IDs never contain commas.
- `pg_conninfo` path silently drops `source_type` and `accepted_pr` — callers targeting PG should not assume metadata parity with SQLite.
- `net.add_node` must tolerate (or reject) duplicate IDs — `add_node` itself performs no existence check.
- Callers must supply antecedents/outlist IDs that already exist (or accept dangling references) — the function does not verify them.

## Usage Patterns

Typical patterns:

```python
# Premise
add_node("net-http-2", "HTTP/2 uses multiplexed streams")

# Derived (SL) from two antecedents
add_node("api-latency-issue", "...", sl="net-http-2,load-balancer-config", source_type="derived")

# Outlist-gated premise
add_node("retry-allowed", "...", unless="circuit-breaker-open")

# Any-mode OR
add_node("service-healthy", "...", sl="probe-a,probe-b", any_mode=True)

# Namespaced belief (auto-creates "agentX:active")
add_node("profile-cache-lru", "...", namespace="agentX")
```

**Caller obligations**
- Provide `source_type` only when it is a valid `SOURCE_TYPES` member.
- Do not assume `any_mode` is layered on top of `namespace` correctly (see above).
- For human-auditable additions, populate `source`, `source_url`, and `source_type`, and consider `access_tags` for downstream access control.

## Dependencies

- `_with_network` (module-local): transaction + cache manager.
- `Justification` (re-exported from `reasons/__init__`): value object for SL/CP/outlist.
- `_pg_dispatch` → `reasons/pg.py:PgApi.add_node`: PG backend.
- `_resolve_namespace` (module-local): namespace-prefix helper.
- `network.add_node`: actually adds the node to the graph and recomputes truth values.
- `storage.Storage` (via `_with_network`): SQLite persistence.
- `pubsub` (via `_with_network.__exit__`): change notification.
- `SOURCE_TYPES` constant (module-level).

---

## Topics to Explore

- [function] `reasons/api.py:_with_network` — Understand the exact write/save/publish lifecycle and the mtime-based cache, which explains when DDL/persistence and pubsub events actually fire.
- [file] `reasons/network.py` — Inspect `Network.add_node` for ID-collision semantics, dependents-index registration, and how truth values propagate (this is where the real TMS work happens).
- [file] `tests/test_any_mode.py` — Confirms whether the "only first justification gets `{ns}:active`" behavior is intentional or untested.
- [file] `reasons/pg.py` — Compare the PG implementation against the SQLite branch to verify metadata parity for `source_type` and `accepted_pr`.
- [general] `namespace-isolation` — How `{namespace}:active` is used as a global retraction lever for a whole namespace, and what the intended OR-mode semantics should be.

## Beliefs

- `add-node-requires-comma-separated-ids` — `sl`, `cp`, and `unless` are parsed as comma-separated strings with no escaping; IDs containing commas cannot be expressed.
- `add-node-any-mode-produces-or-justifications` — When `any_mode=True` and `sl` contains more than one antecedent, one single-antecedent SL justification is created per antecedent.
- `add-node-validates-source-type` — `source_type` must be a member of `{"code", "document", "self-description", "derived"}` or `ValueError` is raised.
- `add-node-pg-drops-source-type-and-accepted-pr` — The PostgreSQL dispatch path omits `source_type` and `accepted_pr`, so those metadata fields are never persisted on PG.
- `add-node-writes-through-with-network` — Every write goes through `_with_network(write=True)`, which skips `save()` when an exception propagates and publishes pubsub change events on success.
