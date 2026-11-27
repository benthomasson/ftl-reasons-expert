---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Function: import_json in reasons/pg.py

# `PgApi.import_json` — Import a network snapshot

**Location:** `reasons/pg.py:882` (approximately)

## 1. Purpose

`import_json` repopulates a single `domain_id`'s belief network from a parsed JSON snapshot — the structure returned by `PgApi.export_network` (`{"meta": …, "nodes": …, "nogoods": …, "repos": …}`). It exists to support round-tripping, backup restoration, and copying a network into a fresh tenancy / domain.

Unlike `import_beliefs` (markdown) or `import_agent` (namespaced agent merge), this path is **the structural JSON importer** and tries to reproduce the exported state *exactly* — including truth values, timestamps, and `_retracted` markers — rather than recomputing justified state from scratch.

## 2. Contract

**Preconditions (unenforced):**
- `data` is a mapping-like object (calls `data.get(...)`). Nodes must be a `dict` of `nid → node_dict`.
- Each `node_dict` is expected to have: `text` (str), optional `truth_value` ∈ `{"IN", "OUT"}`, optional `justifications` (list of `{type, antecedents, outlist, label?}`), optional `metadata` (dict), optional source fields, optional timestamps.
- `self.conn` is an open psycopg v3 connection with `init_db()` already run.
- The caller is prepared to have the connection committed multiple times during this call.

**Postconditions:**
- Every node in `data["nodes"]` that is **not already in `rms_nodes`** for `self.domain_id` is inserted, with its justifications, in dependency order.
- Meta keys `schema_version`, `project_name`, `created_at` from `data["meta"]` are upserted into `rms_network_meta`.
- After `propagate()` recomputes truth, each node's `truth_value` is **forced back** to the export value (`IN` unless explicitly `OUT`), and `_retracted` is added/removed to match.
- Timestamps `updated_at`, `reviewed_at`, `verified_at`, `retracted_at` are restored verbatim from export data (when non-empty).
- Missing nogoods (by `id`) are inserted.
- Multiple `COMMIT`s are issued, so partial success is possible on error.

**Invariants assumed:**
- Referenced nodes in `antecedents` / `outlist` that appear in the import data are inserted first (topological order).
- Referenced nodes that are *not* in the import data are assumed to already exist in the domain. If they don't, `_compute_truth` treats them as `OUT` (silent).

## 3. Parameters

| Name | Type | Notes |
|---|---|---|
| `data` | `dict` | Parsed `export_network()` output. Only `nodes`, `meta`, `nogoods` are read; `repos` is ignored. `data["nodes"]` must be a dict; `data["nodes"]` is iterated twice (once for `remaining`, once for the truth override). |

Edge cases:
- `data.get("nodes")` returns `None` → `dict(None)` raises `TypeError`.
- `data.get("nodes")` returns a list of IDs → `dict(list)` misbehaves (raises `ValueError` for wrong-shaped tuples).
- A node's `truth_value` missing → defaults to `"IN"`.
- A node's `metadata` key value `None` → `None.items()` raises `AttributeError` (the slice `{k: v for k, v in ... .items() if ...}` assumes dict).
- A node's `justifications[i]` missing `antecedents`/`outlist` → defaults to `[]` via `.get`.
- Nogood missing `id` → `KeyError` (uses bracket access, not `.get`).

## 4. Return Value

```python
{"nodes_imported": int, "nogoods_imported": int}
```

- `nodes_imported` counts nodes actually `INSERT`ed (not nodes present, not skipped).
- `nogoods_imported` counts new nogoods inserted (existing IDs are skipped silently).
- **Not returned** (but computed internally): `skipped`, truth-value flips from propagation, restored timestamps, meta changes. This makes the result lossy — the caller cannot tell whether the returned state matches `data`.

Caller must handle:
- Partial imports if an exception hits between commits.
- Idempotent-on-structure: calling twice returns `nodes_imported == 0` on second call, but still re-writes timestamps.

## 5. Algorithm

**Phase 0 — meta.** For each of `schema_version`, `project_name`, `created_at` that is present and truthy in `data["meta"]`, `INSERT … ON CONFLICT (key, domain_id) DO UPDATE SET value = EXCLUDED.value`. Then `commit()`.

**Phase 1 — topological node import.**
- Snapshot the set of already-existing node IDs as `added`.
- Compute `all_node_ids` from the import data.
- Loop up to `len(remaining)+1` passes:
  - For each pending node:
    - Skip if `nid in added` (increments `skipped`, dead counter).
    - Gather `all_deps` from every justification's `antecedents ∪ outlist`.
    - Filter to `deps_in_data` (deps that also appear in the import).
    - If **all** `deps_in_data` are already in `added`: call `_add_node_raw` with the node's fields, strip `_…` metadata keys, mark as added.
    - Else defer to `next_remaining`.
  - If `next_remaining` has the same size as before: **no progress → force-insert all remaining nodes** (handles cycles / unresolvable deps), then `break`.
- `commit()`.

**Phase 2 — fix-up.**
- `self.propagate()` recomputes every derived node to a fixpoint (skips nodes flagged `_retracted`).

**Phase 3 — exact-state override.**
- For each node in `data["nodes"]`:
  - Read current truth value.
  - If it differs from `ndata.get("truth_value", "IN")`:
    - `OUT` target → set `truth_value='OUT'` and `metadata || '{"_retracted": true}'`, log `retract`.
    - `IN` target → set `truth_value='IN'` and `metadata - '_retracted'`, log `assert`.
  - Unconditionally (when non-empty): overwrite `updated_at`, `reviewed_at`, `verified_at`, `retracted_at` from `ndata`.

**Phase 4 — nogoods.**
- For each nogood: skip if `(id, domain_id)` exists; otherwise insert.
- `commit()`.

## 6. Side Effects

- **Inserts** into `rms_nodes`, `rms_justifications`, `rms_network_meta`, `rms_nogoods`.
- **Updates** node `truth_value`, `metadata._retracted`, and four timestamp columns.
- **Recomputes** all derived truth values (via `propagate`), logging to `rms_propagation_log`.
- **Emits log entries**: `"retract"`, `"assert"`, `"propagate"`, `"recompute"`, `"add"` (from `_add_node_raw`).
- **Commits the connection 3–4 times**: after meta, after node inserts, inside `propagate`, and at the end. **Not atomic.**
- Does **not** touch `rms_network_meta` for `updated_at`, `node_count`, or `repo:*` keys — so those drift from the source.
- Does **not** re-register `repos` (present in `export_network` output but ignored here).

## 7. Error Handling

- **Silent / swallowed:**
  - Missing external reference (dep in justification not in data and not in DB) → node inserted with a dangling JSONB reference; `_compute_truth` treats it as `OUT`; only observed on later `propagate`.
  - Existing nodes: text/justifications/metadata are **not** updated — silently discarded on import.
  - Duplicate nogood IDs: silently skipped (existing content untouched).
  - `skipped` counter is computed but never surfaced.

- **Raised (delegated):**
  - `psycopg` exceptions from INSERT/UPDATE (constraint violations, serialization, etc.). There is **no try/except**, so on error:
    - The connection is left mid-transaction; earlier `commit()`s survive, later changes roll back (if the caller closes/rolls back).
  - `TypeError` / `AttributeError` / `KeyError` from malformed input (not wrapped in validation).
  - Propagate errors bubble through unchanged.

There is **no input validation** and no transaction rollback wrapper — this is a sharp-edged function.

## 8. Usage Patterns

```python
data = source_api.export_network()
target = PgApi(conninfo, target_domain_id)
target.init_db()
try:
    result = target.import_json(data)
except Exception:
    target.conn.rollback()
    raise
```

Callers should:
- Wrap in a try/except and roll back on failure (partial commit is a real risk).
- Know that import is a **state reset for truth values** — any OUT/IN the user toggled after the export will be undone by Phase 3.
- Not assume existing nodes are updated — the importer is additive-only for content.

## 9. Dependencies

- Internally: `self._add_node_raw` (node + justifications), `self.propagate` (fixpoint), `self._log` (propagation log), `self.domain_id`, `self.conn`.
- Tables: `rms_nodes`, `rms_justifications`, `rms_nogoods`, `rms_network_meta`, `rms_propagation_log`.
- External: `json.dumps`; psycopg v3 connection (implicitly, the schema in `SCHEMA` must be initialized).
- Complements `export_network` — but only partially: `repos`, `node_count`, `updated_at` are lost.

## Assumptions not enforced by types

- `data["nodes"]` is a `dict[nid, dict]`, not a list.
- Every node's `metadata` value is a `dict` (or absent), never `None`.
- Every justification's `antecedents`/`outlist` are lists of strings (`.update(dict)` on a dict value silently iterates keys).
- Every nogood dict has an `"id"` string key.
- Existing DB nodes with matching IDs are acceptable to leave untouched; the caller doesn't need to reconcile them.
- Domain ID `self.domain_id` is trusted to be a valid UUID string (no validation happens before SQL).

---

## Topics to Explore

- [function] `reasons/pg.py:export_network` — The counterpart producing `data`; check exactly which keys it emits and which `import_json` ignores (repos, meta.updated_at).
- [function] `reasons/pg.py:propagate` — Runs between insert and override; the only place where truth values get recomputed and why the override phase is needed at all.
- [function] `reasons/pg.py:_add_node_raw` — The single insertion primitive; knows how `_compute_truth` seeds initial truth from justifications, which explains why the fallback/cycle path is safe.
- [file] `tests/test_import_json.py` — Validates round-trip fidelity; the tests reveal which invariants are enforced vs assumed.
- [general] `import-path-comparison` — Compare `import_json`, `import_beliefs`, and `import_agent` to see how different importers treat idempotency, cycles, and namespacing differently.

## Beliefs

- `import-json-forces-export-truth` — After `propagate`, every node's `truth_value` is overwritten to match the export, so user toggles since export are undone.
- `import-json-skips-existing-node-content` — Nodes already present in `rms_nodes` are not updated for text, justifications, or metadata; only truth and timestamps can change.
- `import-json-has-partial-commit-window` — The function issues multiple `COMMIT`s, so a mid-call exception can leave the domain in a partially imported state.
- `import-json-drops-repos-and-some-meta` — `repos`, `node_count`, and `updated_at` meta keys from `export_network` are not re-imported.
- `import-json-silently-allows-dangling-refs` — Justifications referencing nodes absent from both the import and the target domain insert successfully because the FK is on `rms_justifications.node_id`, not on JSONB antecedent content.
