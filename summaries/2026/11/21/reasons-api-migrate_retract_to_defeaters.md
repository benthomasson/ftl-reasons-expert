---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: migrate_retract_to_defeaters in reasons/api.py

# `migrate_retract_to_defeaters`

## Purpose

This is a **one-shot data migration helper** that converts the legacy "string-based retraction" mechanism into the graph-native defeater mechanism.

Historically, retracting a node (`retract_node(node_id, reason=...)`) set `metadata["retract_reason"]` and, in the TMS recompute pass, a `_retracted` flag in metadata would force the node's truth value to `OUT` regardless of whether its justifications were still satisfied. That is a side-channel out of band from the network's justification graph.

The graph-native way to do the same thing is to add a *defeater* node to the justification's `outlist`. A defeater is a premise (no justifications → `IN`), and while it's `IN` the justification is blocked. This is the same mechanism `defeat_justification` and `defeat_with_scope` use.

`migrate_retract_to_defeaters` finds nodes that are currently kept `OUT` *only* by the legacy `retract_reason` string, creates a defeater premise for the first justification that *would* otherwise be satisfied, drops the string metadata, and lets normal TMS recomputation preserve the `OUT` state via the outlist.

## Contract

**Preconditions**
- A valid RMS SQLite database exists at `db_path`.
- If `node_ids` is provided and non-empty, its entries are expected to be node IDs; unknown IDs are tolerated (reported as errors).
- `node.metadata` is assumed to be a mutable `dict` (this is not enforced — see assumptions).

**Postconditions (when `dry_run=False`)**
- For every migrated node:
  - `metadata["retract_reason"]` and `metadata["_retracted"]` are removed.
  - A new premise node `migrated-retraction-{nid}-j{idx}` exists with metadata `defeater_type="migrated-retraction"`, `defeats_node=nid`, `defeats_justification=idx`.
  - That defeater ID is appended to the target justification's `outlist`.
  - `defeater.dependents` contains the target `nid` (reverse index updated manually).
  - The target node's `updated_at` is refreshed.
- `net.recompute_all()` is invoked once at the end if anything was migrated, so truth values are consistent with the new graph.
- The database file is saved and the `_network_cache` entry is refreshed by `_with_network`. If `pubsub.has_subscribers(db_path)` is true, change events are published.

**Postconditions (when `dry_run=True`)**
- No database mutation; read-only access via `_with_network(write=False)` (which will use the cached `Network` when the file mtime is unchanged).
- Returned report describes what *would* be migrated.

**Invariants preserved by the function**
- A migrated entry always carries `id`, `retract_reason`, `justification_index`; applied entries additionally carry `defeater_id`.
- `skipped` entries always have `id` + `reason` from a known enum (`"premise (no justifications)"`, `"no satisfied justification"`).
- `errors` entries always have `id` + `reason` (`"not found"` for unknown node IDs).

## Parameters

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `node_ids` | `list[str] \| None` | `None` | Restrict the migration to these IDs. **Caution:** `[]` is falsy — the expression `node_ids or list(net.nodes.keys())` treats an empty list identically to `None` (migrate *all* nodes). |
| `dry_run` | `bool` | `True` | If `True`, this is a preview only; the DB is opened read-only, no nodes/justifications/metadata are modified, and `recompute_all` is not called. Passing `False` enables writes. Safe-by-default. |
| `db_path` | `str` | `DEFAULT_DB` (`"reasons.db"`) | Path to the SQLite database. |

## Return Value

```python
{
    "migrated": [ ... ],   # items that were (or would be) converted
    "skipped":  [ ... ],   # items with retract_reason that couldn't be converted
    "errors":   [ ... ],   # items that couldn't be inspected (missing IDs)
}
```

Concrete shapes:

- **`migrated`**
  - Dry run: `{"id", "retract_reason", "justification_index"}`
  - Applied: `{"id", "defeater_id", "retract_reason", "justification_index"}`
- **`skipped`**: `{"id", "reason"}` — reasons are exactly `"premise (no justifications)"` or `"no satisfied justification"`.
- **`errors`**: `{"id", "reason": "not found"}` — the only current reason string.

The result dict is JSON-serializable, matching the module's design contract. Nodes that are `IN`, or that have no `retract_reason` at all, are **silently omitted** — they appear in neither `migrated`, `skipped`, nor `errors`. This is important: `skipped` is *not* "everything we didn't migrate", it is "everything with legacy retraction metadata we chose not to convert".

## Algorithm

1. **Open the network.** `_with_network(db_path, write=not dry_run)` — for dry run, reuse the cached `Network` if mtime matches; for writes, always load fresh. On exit, save (write mode) and republish events.
2. **Build the candidate list.** `node_ids or list(net.nodes.keys())` — falls back to all nodes for `None` and `[]`.
3. **For each `nid`:**
   1. If `nid not in net.nodes`: append `{"id", "reason": "not found"}` to `errors`, `continue`.
   2. If `node.truth_value != "OUT"`: `continue` **silently** (already `IN`, not a retracted node).
   3. Read `retract_reason = (node.metadata or {}).get("retract_reason")`. If missing: `continue` silently.
   4. If `not node.justifications` (premise): `skipped += {"id", "reason": "premise (no justifications)"}`, `continue`.
   5. **Find first satisfied justification** (`satisfied_idx`):
      - All antecedents are present and `IN`.
      - All outlist entries are either absent from the network or `OUT`.
      - Break on first match.
   6. If `satisfied_idx is None` → the node would be `OUT` even without the `_retracted` flag. Skip with `"no satisfied justification"`, leave metadata untouched (so a future pass can retry when the world state changes).
   7. **Dry run branch.** Append the preview entry to `migrated`; do not mutate.
   8. **Apply branch:**
      - `defeater_id = f"migrated-retraction-{nid}-j{satisfied_idx}"`.
      - `defeater_text = f"{retract_reason} (defeats {nid} justification {satisfied_idx})"`.
      - Create the defeater via `net.add_node` with metadata tagging `defeater_type`, `defeats_node`, `defeats_justification`. No justifications → premise → `IN`.
      - Append `defeater_id` to `node.justifications[satisfied_idx].outlist` (idempotent guard `if defeater_id not in …`).
      - `defeater.dependents.add(nid)` — **manual reverse-index upkeep**. The network does not auto-maintain `dependents` when a justification's outlist is mutated by the API layer.
      - Bump `node.updated_at`.
      - `node.metadata.pop("retract_reason", None)` and `node.metadata.pop("_retracted", None)`.
      - Append the applied entry (with `defeater_id`) to `migrated`.
4. **Recompute (apply only, and only if anything changed).** `if not dry_run and migrated: net.recompute_all()`. This is where the target nodes are re-derived: the new defeater is `IN`, so the corresponding justification is now blocked and the node stays `OUT`.
5. **Return** the three lists.

## Side Effects

- **Writes** (only when `dry_run=False`):
  - `Storage.save(network)` on context exit.
  - `_network_cache[db_path]` updated to `(mtime, network)`.
  - `pubsub.compute_changes` / `pubsub.publish` fire if subscribers exist.
  - New nodes, mutated `outlist` lists, mutated metadata, and `updated_at` on target nodes.
- **Reads** (always): read `nodes`, `justifications`, `metadata`, and network invariants for the satisfied-justification check.
- `dry_run=True` still opens the SQLite file to read the mtime but does not write it.

## Error Handling

- `KeyError` is **not** raised for unknown `node_ids` — they end up in `errors`.
- `FileExistsError`/`OSError`/`sqlite3.*` from `_with_network`'s `Storage(db_path)`/`load()` are **not** caught — a corrupt or missing DB propagates.
- `net.add_node` errors (e.g., ID collision — see assumptions) are **not** caught and will propagate, aborting the whole migration *after* some nodes may already have been mutated in memory. Because `_with_network.__exit__` only calls `save` when `exc_type is None`, an exception mid-loop means **no partial writes** are saved. So it's effectively transactional across the whole batch.
- `node.metadata` being `None` would cause `AttributeError` on `node.metadata.pop(...)` inside the apply branch, because that call lacks the `node.metadata or {}` guard used elsewhere in this file. This is a latent assumption in the code.

## Usage Patterns

Typical flow during a version upgrade:

```python
# 1. Preview
plan = migrate_retract_to_defeaters(dry_run=True)
print(len(plan["migrated"]), "would migrate;", plan["skipped"], plan["errors"])

# 2. Restrict to specific nodes if desired
migrate_retract_to_defeaters(
    node_ids=["old-claim", "stale-doc"],
    dry_run=False,
)

# 3. Apply everything
result = migrate_retract_to_defeaters(dry_run=False)
```

Caller obligations:
- Inspect `errors` — missing node IDs suggest typos or nodes deleted between plan and apply.
- Inspect `skipped` with reason `"no satisfied justification"`: those nodes still have legacy metadata and will need another migration pass once their antecedents become `IN`.
- Because `skipped` only includes nodes with `retract_reason`, do not use it as an audit of "what was ignored" — it excludes `IN` nodes and nodes without the string.

Idempotency: after a successful apply, the target nodes no longer have `retract_reason`, so a second run will classify them as silently irrelevant. The function is therefore safe to re-run.

## Dependencies

- **`_with_network(db_path, write=...)`** — local helper that owns `Storage` and `Network` lifetimes, mtime-keyed caching, saving, and `pubsub` event emission.
- **`Network.add_node`**, **`Network.recompute_all`**, **`Network.nodes`** — from `reasons/network.py`.
- **`Node.justifications[i].outlist`**, **`Node.dependents`**, **`Node.metadata`**, **`Node.truth_value`**, **`Node.updated_at`** — direct attribute access on nodes.
- **`Justification`** — only via attribute access (`.antecedents`, `.outlist`); no instantiation here.
- **Standard library only**: `os.path.getmtime` (inside `_with_network`), `sqlite3`, `datetime`.
- `DEFAULT_DB = "reasons.db"` is the implicit default path.

## Assumptions Not Enforced by the Type System

These are the load-bearing assumptions that the type hints (`list[str] | None`, `bool`, `str`) do not capture:

1. **`retract_reason` in metadata implies `_retracted` was set** — i.e., the string is a marker of legacy retraction, not arbitrary user metadata by that key. A caller that stored a *reason for future retraction* under `retract_reason` would get surprise conversions.
2. **`_retracted` is a distinct legacy key** the pop cleans up. If it lives somewhere else (e.g., `node.state`), the pop is a no-op and the migration would convert while leaving the `_retracted` flag — recompute would still force `OUT` for a different reason. (Worth verifying against `network.py`.)
3. **`defeater_id` collisions are impossible.** No check is made before `net.add_node(id=defeater_id, ...)`. If a node named `migrated-retraction-{nid}-j{idx}` already exists, behavior depends on `Network.add_node` (overwrite vs. raise) and is not handled here.
4. **`Node.dependents` must be manually updated when appending to `outlist`.** This mirrors `defeat_justification`. Callers that append to `outlist` without touching `dependents` risk inconsistent traversals (`_cascade_depth`, `_rewrite_dependents`).
5. **`node.metadata` is a dict, not `None`.** The apply branch does `node.metadata.pop(...)` without the `or {}` guard prevalent elsewhere in this file.
6. **`net.recompute_all()` is safe to call after partially-mutated memory state.** It's only invoked once, after the loop, so if `recompute_all` throws, `_with_network.__exit__` will *not* save (because an exception was raised) — auto-rollback semantics, not enforced by type.
7. **`node_ids=[]` means "all nodes"** because of `node_ids or list(net.nodes.keys())`. If a caller intended "no nodes", they'd get a full migration.
8. **A defeater premise node stays `IN`** by default (undefined `truth_value` from `add_node` yields `IN`). This is what preserves the target's `OUT` state, but it's an emergent property of `Network.add_node`, not a documented contract here.

## Topics to Explore

- [file] `reasons/network.py` — Inspect `Network.retract` and `Network.recompute_all` to confirm how `_retracted` and `retract_reason` are consumed, and whether the migration is still needed after recent versions.
- [function] `reasons/api.py:defeat_justification` — Sibling function; the inlined defeat logic here exists only because `_with_network` contexts can't nest, so this is the canonical reference implementation of the same "add defeater to outlist" operation.
- [file] `tests/test_migrate_defeaters.py` — Covers the concrete preconditions/postconditions (missing IDs, unsat justifications, dry-run vs apply, idempotency); read before touching this function.
- [function] `reasons/network.py:Network.add_node` — Determines what happens on a `defeater_id` collision, which the migration currently trusts never happens.
- [general] `_retracted-flag-lifecycle` — Trace where `_retracted` is read and written across the codebase; it underpins the "would be IN if not for `_retracted`" premise of this migration.

## Beliefs

- `migrate-retract-to-defeaters-is-idempotent` — After a successful apply run, every target loses `retract_reason` metadata, so a second invocation reports zero migrations for those nodes.
- `empty-node-ids-migrates-all-nodes` — Because the candidate expression is `node_ids or list(net.nodes.keys())`, passing `node_ids=[]` is equivalent to passing `None` and migrates the entire network.
- `migration-requires-defeater-dependents-update` — The function must call `defeater.dependents.add(nid)` after appending to `justification.outlist`, because the network does not auto-maintain the reverse index on outlist mutation.
- `skipped-list-excludes-in-and-metadata-free-nodes` — `skipped` only contains nodes that had `retract_reason` but were rejected (premise input or no satisfied justification); `IN` nodes and nodes without `retract_reason` are omitted from all three return lists.
- `dry-run-uses-cached-network-read-only` — With `dry_run=True`, `_with_network(db_path, write=False)` reuses the mtime-keyed `_network_cache` and never saves or publishes events.
