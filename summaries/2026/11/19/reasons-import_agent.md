---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/import_agent.py

# `reasons/import_agent.py`

## Purpose

This module owns **cross-agent belief federation** for the Reason Maintenance System (RMS). It lets one network pull in another agent's beliefs as a *namespaced subnetwork* (`agent_name:belief-id`) while retaining the ability to globally retract that agent's entire belief set with a single action.

Its two responsibilities are:

1. **Import** (`import_agent`, `import_agent_json`) — one-shot insertion of a foreign agent's beliefs. Existing nodes are skipped.
2. **Sync** (`sync_agent`, `sync_agent_json`) — idempotent reconciliation with remote-wins semantics: additions, updates, retractions of remote-deleted nodes, and resurrection of previously-retracted nodes.

Both come in **Markdown** and **JSON** flavors; JSON is the lossless format (preserves justifications and outlists on OUT nodes), while Markdown intentionally flattens OUT/STALE beliefs to bare premises.

---

## Key Components

### Agent-safety scaffolding

- **`_ensure_agent_nodes(network, agent_name, source_path)`**
  Creates two infrastructure nodes if missing:
  - `<agent>:active` — a premise node (no justifications) representing trust in the agent.
  - `<agent>:inactive` — a relay node with `Justification(type="SL", antecedents=[], outlist=[<agent>:active])`.
  Returns `(active_id, inactive_id, created_premise)`.

  **Contract:** if `active` is retracted, `inactive` becomes IN; because every imported belief has `inactive` in its outlist, all get cascaded OUT. This is the *"kill switch."*

- **`_build_justifications(claim, prefix, inactive_id, agent_name)`**
  Transforms normalized `raw_justifications` into `Justification` objects:
  - Prefixes every antecedent and outlist entry with `agent_name:`.
  - **Prepends `inactive_id` to every outlist** — this is how the kill switch bites each imported belief.
  - Supplies a fallback bare-premise justification (`antecedents=[], outlist=[inactive_id]`) when a claim has no justifications.

### Normalization layer

Converts input formats into a uniform claim shape:
```
{id, text, is_out, source, [source_url,] source_hash, date,
 metadata, raw_justifications: [{type, antecedents, outlist, label}]}
```
Where `raw_justifications` uses **unprefixed** IDs.

- **`_normalize_markdown(beliefs_text, only_in)`** — calls `parse_beliefs`; filters `IN`-only if requested; statuses `STALE`/`OUT` → `is_out=True`; strips dependencies into a single SL justification for IN nodes, filtered to only-interior references (dangling refs are dropped); drops justifications entirely for OUT nodes.
- **`_normalize_json(data, only_in)`** — same idea for `data["nodes"]`; **preserves the full justification list** including outlists, filtered to interior references.
- **`_normalize_nogoods_markdown` / `_normalize_nogoods_json`** — thin adapters over `parse_nogoods` / JSON `nogoods` key.

### Shared graph operations

- **`_topo_sort_claims(claims)`** — iterative, bounded passes ensuring antecedents are imported before dependents. Cycles are handled by appending the un-orderable remainder after `max_passes`; caller relies on `Network.add_node` tolerating forward references resolved via `_rebuild_dependents`.
- **`_import_nogoods(network, prefix, nogoods)`** — appends `Nogood` objects **only if** at least two prefixed nodes exist and the prefixed nogood id is new.
- **`_fixup_dependents(network)`** — calls the private `network._rebuild_dependents()` because nodes might reference outlist targets that didn't exist at creation time.
- **`_justifications_match` / `_update_node_justifications`** — exact equality check (type, antecedents, outlist, label); update path always rebuilds dependents.

### Import vs Sync cores

- **`_import_claims`** — skips nodes whose prefixed id already exists; otherwise adds. Distinguishes three OUT cases:
  1. `is_out` and no justifications (markdown path) → queue into `retract_after`, retracted **after** `recompute_all`.
  2. `is_out` with justifications (JSON path) → set `metadata["_retracted"] = True` and `truth_value = "OUT"` directly.
  3. Else (IN) → normal add.
  Returns a stats dict (`claims_imported`, `claims_skipped`, `claims_retracted`, `claims_propagated`, `nogoods_imported`, etc.).

- **`_sync_claims`** — remote-wins reconciliation:
  - Existing nodes: field-by-field diff on `text/source/source_url/source_hash/date/metadata`; rebuild justifications and compare; set/clear `_retracted`; queue transitions through `assert_after` (resurrections) and `retract_after` (retractions).
  - Nodes present locally but absent from remote → `network.retract(...)` if still IN, else set `_retracted` (idempotency).
  - Order: fixups → assert_after → `recompute_all` → retract_after → nogoods.
  - Returns distinct stats: `beliefs_added`, `beliefs_updated`, `beliefs_removed`, `beliefs_retracted`, `beliefs_unchanged`, etc.

### Public API

| Function | Format | Mode |
|---|---|---|
| `import_agent` | markdown | import |
| `import_agent_json` | JSON | import |
| `sync_agent` | markdown | sync |
| `sync_agent_json` | JSON | sync |

All accept `only_in` (drop STALE/OUT sources) and return the stats dict.

---

## Patterns

- **Normalize-then-process** — the four `_normalize_*` functions flatten format differences so that import and sync can run identical business logic.
- **Namespacing via prefix** — every created id is `f"{agent_name}:{local_id}"`. Justifications are rewritten in `_build_justifications` so all internal refs stay inside the agent's namespace.
- **Kill-switch via outlist, not antecedent** — the docstring is explicit: `active` is *not* an antecedent (that would give every belief an always-valid fallback defeating per-belief retraction); only `inactive` appears in outlists.
- **Two-phase retract/propagate** — `retract_after` (and `assert_after` for sync) queues truth-value flips *after* `_fixup_dependents` + `recompute_all`, ensuring the dependency graph is fully built before propagation.
- **Sentinel metadata (`_retracted`)** — a per-node flag that `recompute_all()` and `_propagate()` respect, keeping nodes OUT until explicitly cleared.
- **Idempotency counters** — `beliefs_unchanged` / `claims_skipped` reflect the diff-based design.
- **Deterministic iteration** — `for node_id in sorted(removed_ids)` guarantees stable order of removals.

---

## Dependencies

**Imports:**
- `pathlib.Path` — used only to resolve `network.repos[agent_name] = str(Path(source_path).resolve().parent)`.
- `from . import Justification, Nogood` — network primitives (re-exported by `reasons/__init__.py`).
- `from .import_beliefs import parse_beliefs, parse_nogoods` — markdown parsers.
- `from .network import Network` — type for public signatures and calls `add_node`, `retract`, `assert_node`, `recompute_all`, `_rebuild_dependents`.

**Imported by:**
- `reasons/api.py` — likely exposed through the top-level `api` surface.
- `reasons/pg.py` — for persistence/dispatch to Postgres.
- `reasons_lib/import_agent.py` — mirror copy (repo has parallel `reasons/` and `reasons_lib/` trees).

**Tight coupling note:** reaches into `Network` privates — `_rebuild_dependents`, `network.nodes[...].truth_value`, `network.nodes[...].justifications`, `network.repos`.

---

## Flow

### `import_agent(...)` (markdown)

1. `_normalize_markdown` → claims.
2. `_normalize_nogoods_markdown` → nogoods.
3. `_import_claims`:
   1. `_ensure_agent_nodes` (creates `active`/`inactive` if missing).
   2. Record `network.repos[agent_name]`.
   3. `_topo_sort_claims`.
   4. For each claim: skip if `agent:id` already in `network.nodes`; else build justifications + metadata, `add_node`, count. OUT handling per case above.
   5. `_import_nogoods`.
   6. `_fixup_dependents` then `recompute_all()` to compute initial truth values.
   7. Process `retract_after` (call `network.retract`).
   8. Return stats.

### `sync_agent(...)` (markdown)

Same normalization, but `_sync_claims`:
1. `_ensure_agent_nodes`.
2. Compute `remote_ids`, `infra_ids = {active, inactive}`, `local_agent_ids` (prefixed minus infra).
3. `_topo_sort_claims`.
4. For each claim: add or diff-update, with is_out/justs branching builds `assert_after` and `retract_after`.
5. Remove `local_agent_ids - remote_ids`: retract if IN, else mark `_retracted`.
6. `_fixup_dependents`.
7. Run `assert_after` (pop `_retracted`, `assert_node`).
8. `recompute_all`.
9. Run `retract_after` (retract IN nodes; `_retracted` their justifi-holding counterparts).
10. Import nogoods; return stats.

---

## Invariants

- **Kill switch is universal:** every imported belief's every justification has `inactive_id` in its outlist (set unconditionally in `_build_justifications`).
- **`active_id` never appears in antecedents:** the docstring insists on this; `_build_justifications` only ever inserts `inactive_id`, never `active_id`.
- **Justification references are namespaced and interior:** antecedents/outlist entries are filtered in the normalizers to ids present in the claim set (no dangling refs after prefixing).
- **Antecedents precede dependents in insertion order** (best effort via topo sort; cycles appended).
- **`_retracted` blocks recomputation:** nodes carrying `_retracted=True` do not become IN through `_propagate`/`recompute_all`, keeping remote-OUT nodes authoritative until the remote says IN.
- **Nogoods require ≥2 valid prefixed nodes** and a fresh prefixed id.
- **`recompute_all()` is called before any queued retract** in `_import_claims`, so dependents are established first.
- **`import` never mutates existing nodes**; only `sync` does.
- **`local_agent_ids` excludes `active`/`inactive`** so the infrastructure nodes are never treated as deletions in sync.

---

## Error Handling

There is **no try/except and no validation raising** in this module. It is a "let downstream raise" design:

- Malformed markdown/JSON propagates from `parse_beliefs`, `parse_nogoods`, or dictionary access (`data["nodes"]` assumes key exists; `ndata.get(...)` for values).
- Missing/incorrectly-typed keys surface as `KeyError`/`AttributeError` from `network.add_node` or the Justification/Nogood constructors.
- `network.retract` / `assert_node` on unknown nodes would propagate from `Network`.
- The only defensive check is `_import_nogoods`'s id-collision and `len(valid_nodes) >= 2` guard.
- Cycle handling in `_topo_sort_claims` is a graceful degrade (append leftover), not an error.
- Note: `assert_after` is a **data structure** (list of node ids to re-assert IN), *not* a Python `assert`. `network.assert_node(...)` is the RMS operation "make this node IN," not a debug assertion.

---

## Topics to Explore

- [file] `reasons/network.py` — Where `add_node`, `retract`, `assert_node`, `recompute_all`, and `_rebuild_dependents` are defined; essential to understand `_retracted`-aware propagation.
- [file] `reasons/import_beliefs.py` — Source of `parse_beliefs`/`parse_nogoods`; defines the markdown format the normalizers consume.
- [file] `reasons/pg.py` — How imported `network.repos[agent_name]` and stats dicts are persisted; likely where transaction boundaries sit.
- [function] `reasons/network.py:_propagate` — The truth-value semantics (IN/OUT/STALE) that `_retracted` interacts with.
- [file] `tests/test_import_agent.py` — Behavioral contracts (kill switch, JSON resurrection, idempotency) encoded as assertions.

---

## Beliefs

- `inactive-relay-implements-kill-switch` — Every imported belief's every outlist includes `<agent>:inactive`, so retracting `<agent>:active` cascades the entire agent's beliefs OUT.
- `active-premise-is-excluded-from-antecedents` — `<agent>:active` is never placed in antecedents of imported justifications; only `<agent>:inactive` appears in outlists.
- `_retracted-metadata-prevents-resurrection` — Nodes marked `metadata["_retracted"]=True` are skipped by propagation so remote-OUT beliefs stay OUT until an explicit IN is synced.
- `sync-agent-uses-remote-wins-semantics` — `_sync_claims` updates existing node fields, adds remote-only beliefs, and retracts local beliefs absent from the remote.
- `json-import-preserves-out-node-justifications` — `_normalize_json` retains justifications on `truth_value=="OUT"` nodes, enabling resurrection; `_normalize_markdown` strips them.
- `import-agent-skips-existing-node-ids` — `_import_claims` short-circuits with `skipped += 1` when `agent:id` already exists in the network.
