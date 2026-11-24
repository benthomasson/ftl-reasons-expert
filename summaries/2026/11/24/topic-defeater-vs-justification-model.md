---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Understand how `Justification` rows vs. defeater nodes interact (two ways to defeat a belief in this system).

# Justification Rows vs. Defeater Nodes

This system has two distinct persistence representations involved in defeating a belief, and they are wired together by the `outlist` field on each `Justification`:

1. **`justifications` rows** — one row per justification attached to a node; each carries an `outlist` (JSON list of node IDs).
2. **`nodes` rows** — defeater IDs are ordinary node IDs that live in the same `nodes` table.

When a defeater node's truth value changes to `IN`, the justification whose `outlist` contains that ID stops firing; if no other justification of the target node fires, the target goes `OUT`. In parallel, the older *direct retraction* mechanism sets the node itself `OUT`; `migrate_retract_to_defeaters` converts retractions into justify-level defeaters.

---

## 1. The `Justification` row

`Justification` is defined in `reasons/__init__.py:7`. From how it is serialized in `reasons/storage.py` we can infer its shape:

```python
Justification(
    type=jtype,
    antecedents=json.loads(ant_json),
    outlist=json.loads(out_json),
    label=label,
    content_hash=content_hash or "",
)
```

Persistence:

- **SQLite**: `reasons/storage.py` declares
  ```
  CREATE TABLE IF NOT EXISTS justifications (
      rowid INTEGER PRIMARY KEY AUTOINCREMENT,
      node_id TEXT NOT NULL REFERENCES nodes(id),
      type TEXT NOT NULL,
      antecedents_json TEXT NOT NULL DEFAULT '[]',
      outlist_json TEXT NOT NULL DEFAULT '[]',
      label TEXT DEFAULT '',
      content_hash TEXT DEFAULT ''
  );
  ```
- **PostgreSQL**: `reasons/pg.py` mirrors it in `rms_justifications` (with `antecedents JSONB` and `outlist JSONB`) and adds `idx_rms_justifications_outlist` — a GIN index, indicating the `outlist` is queried during propagation.

Save/load (`Storage.save` / `Storage.load` in `reasons/storage.py`) write `json.dumps(j.outlist)` and read `outlist=json.loads(out_json)`. So the `outlist` is the **only** link from a justification to its defeaters — it is a list of node IDs.

The `outlist` (not `antecedents`) is the "defeat" half of the row: antecedents must be `IN` for the justification to fire, outlist members must be `OUT` (i.e., no defeater is present) for the justification to fire. Confirmed indirectly by `test_defeat_justification_added_to_outlist` in `tests/test_defeat_justification.py`, which asserts `result["defeater_id"] in node["justifications"][0]["outlist"]`.

---

## 2. The defeater node

`defeat_justification` in `reasons/api.py:1322` creates a **premise** node in the graph and appends its ID to the target justification's outlist. The docstring says:

> "Defeat a justification by adding a defeater belief to its outlist." (`api.py:1331`)

The key mutations are `api.py:1386–1394`:

```python
defeater = net.add_node(
    id=defeater_id,
    text=defeater_text,
    ...
)
if defeater_id not in justification.outlist:
    justification.outlist.append(defeater_id)
defeater.dependents.add(node_id)
```

- The defeater is a normal node — `test_defeater_in_export` (in `tests/test_defeat_justification.py`) shows it round-trips through `export_network` as `data["nodes"][result["defeater_id"]]`.
- It carries metadata: `defeater_type`, `defeats_node`, `defeats_justification` (asserted in `test_defeat_justification_basic`).
- `defeater.dependents.add(node_id)` registers the defeated node as a *dependent* of the defeater so that propagation reaches the defeated node when the defeater toggles truth value (`test_defeat_restoration_on_retract` verifies retracting the defeater restores the belief).

Default `defeater_id` is built from `defeater_type` + target node + justification index (`api.py:1371–1372`):

```python
defeater_id = f"{defeater_type}-{node_id}-j{justification_index}"
```

So `over-generalizes-belief-j0` means: an `over-generalizes` defeater aimed at `belief`'s justification 0. Well-known `defeater_type`s exercised in `test_defeater_types`: `invalid-inference`, `over-generalizes`, `duplicate-of`, `superseded-by`; plus `migrated-retraction` used by the migration path.

---

## 3. How they interact — the `outlist` bridge

The interaction is asymmetric:

| Where | What | Role |
|---|---|---|
| `justifications.outlist_json` / `rms_justifications.outlist` | list of node IDs | Justification points at its defeaters |
| `nodes` / `rms_nodes` row | a node with a truth value | Defeater's own IN/OUT state |
| `Node.dependents` (in-memory) | set of node IDs | Defeater points at the *defeated node*, so propagation reaches it |

The steps when `defeat_justification` runs:

1. `net.add_node(...)` creates the defeater as a premise node → its `truth_value` defaults `IN`.
2. `justification.outlist.append(defeater_id)` — persists the linkage via `outlist_json` on the justification row.
3. `defeater.dependents.add(node_id)` — sets up propagation.

The belief then goes `OUT` because the justification's `outlist` is non-empty-and-`IN`, so the justification does not fire. This is confirmed by `test_defeat_justification_basic`, which asserts final truth value `OUT` **and** per-justification scoping in `test_defeat_multiple_justifications`:

> Defeat justification 0 → belief stays `IN` (justification 1 still fires).
> Defeat justification 1 → belief finally goes `OUT`.

This per-justification scoping is the key differentiator from direct retraction — see §4.

Restoration is symmetric: `retract_node(defeater_id)` sets the defeater `OUT` → the justification fires again → `test_defeat_restoration_on_retract` asserts the belief returns to `IN`.

---

## 4. The second way: direct node retraction

Even though `defeat_justification` is now the graph-native mechanism, `retract_node` still exists in both backends (`reasons/storage.py`, `reasons/pg.PgApi.retract_node`). It flips the node's own `truth_value` to `OUT` and stamps `_retracted = True` (plus optional `retract_reason`) into metadata — but it does **not** touch any `justification.outlist`.

So the two ways to make a belief `OUT`:

| Mechanism | Scope | Persisted as | Restored by |
|---|---|---|---|
| `retract_node(id, reason=...)` | The **node** itself | `nodes.truth_value = OUT`, `metadata._retracted`, `metadata.retract_reason` | `assert_node(id)` |
| `defeat_justification(id, j_idx, reason, ...)` | One **justification** of the node | new `nodes` row + entry in `justifications.outlist_json` | `retract_node(defeater_id)` |

`migrate_retract_to_defeaters` (tested in `tests/test_migrate_defeaters.py`) recognizes the first form and rewrites it as the second:

- `test_basic_migration`: after `retract_node("derived", reason="Invalid reasoning")`, migration produces defeater `migrated-retraction-derived-j0`, and crucially **`"retract_reason" not in node["metadata"]`** — the `_retracted`/`retract_reason` metadata is removed, replaced by the graph-native defeater.
- `test_skip_unsatisfied_justification`: migration is skipped when the antecedent itself is `OUT` (there's nothing to defeat — the justification already fails on its own antecedents).
- `test_reversibility`: retracting the migrated defeater restores the belief to `IN` — proving the migration is semantically equivalent.
- `test_metadata_cleared` confirms the `_retracted` marker is stripped, so `assert_node` is no longer needed to restore.

The migration test also hints at why the newer path exists: retraction makes the node `OUT` unconditionally, whereas a defeater can be scoped to one justification and can carry structured provenance (`defeater_type`) that downstream tooling can classify.

---

## 5. Two defeater-node flavors (bare premise vs. scope-backed)

Within `defeat_justification` there are two variants declared back-to-back in `reasons/api.py`:

- **Bare premise defeater** — `api.py:1322` (docstring: *"Creates a defeater belief (premise node)"*). The defeater is a leaf premise with no antecedents; retracting it or asserting it directly toggles it.
- **Scope-backed derived defeater** — `api.py:1419` (docstring: *"Defeat a justification with a derived defeater backed by scope beliefs"*). Its docstring (`api.py:1426–1429`) elaborates:

  > "Instead of a bare premise defeater, creates scope beliefs (premises describing what each antecedent establishes) and a derived defeater... causes the defeater to go OUT, restoring the original belief."

So in the scope-backed variant, the defeater node is *derived* from scope premises — retracting a scope premise cascades into the defeater going `OUT`, which in turn restores the defeated belief. Both variants still ultimately go through the same `justification.outlist` bridge; only the anatomy of the defeater node differs.

The system additionally offers `classify_defeat_reason_types` (tested in `tests/test_classify_defeaters.py`) which writes a `defeat_reason_type` metadata key onto existing defeater nodes — 7 fixed types — but does **not** create or alter the outlist. It is orthogonal post-hoc annotation.

---

## 6. What the observations don't show (limits)

The exact evaluation logic that consumes `outlist` at propagation time is **not** in the provided excerpts:

- `reasons/network.py` (in-memory `Network._rebuild_dependents`, propagation loop) is not shown, so I cannot cite exactly how the in-memory engine treats `outlist` — only that `defeater.dependents.add(node_id)` seeds it manually.
- `reasons/pg.py:_compute_truth` / `_propagate` bodies are not shown beyond their call sites; the presence of `idx_rms_justifications_outlist` and `idx_rms_justifications_antecedents` strongly implies both are queried, but the exact SQL is not in the dump.
- `defeat_justification`'s full body (lines ~1348–1410) and the scope-backed variant's full body (lines ~1419+) are only partially captured by search matches.

To fully nail down propagation semantics, those three spots need to be read.

---

## Topics to Explore

- [file] `reasons/api.py` — Read the full bodies of `defeat_justification` (from ~1348) and the scope-backed variant (~1419) to see how defeater metadata is written and how the belief's truth value is re-evaluated immediately after outlist mutation.
- [file] `reasons/network.py` — Inspect `_rebuild_dependents` and the propagation loop; confirm that the `dependents` set is reconstructed from `outlist` on load (otherwise retracting a defeater after reload would not restore the belief).
- [function] `reasons/pg.py:PgApi._compute_truth` — See the concrete SQL that scans `rms_justifications.outlist` and `antecedents` to decide if a justification fires; this is where `idx_rms_justifications_outlist` pays off.
- [function] `reasons/api.py:defeat_justification` — Compare the scope-backed variant with the bare premise variant; note how scope premises feed the defeater's own antecedents, creating a two-level cascade.
- [general] `defeater-classification-pipeline` — The `classify_defeat_reason_types` flow in `reasons/api.py` (`tests/test_classify_defeaters.py`) adds `defeat_reason_type` metadata to existing defeaters; worth understanding as a post-hoc enrichment layer that is independent of the outlist mechanism.

## Beliefs

- `justification-outlist-stores-defeater-node-ids` — Each `Justification` carries an `outlist` of node IDs, persisted as `outlist_json` in SQLite (`reasons/storage.py`) and `outlist JSONB` in PostgreSQL (`reasons/pg.py`); the IDs refer to real rows in the `nodes` table.
- `defeat-justification-creates-a-defeater-node-and-appends-to-outlist` — `defeat_justification` (`reasons/api.py:1322`) calls `net.add_node(...)` to build a defeater node, appends its ID to the targeted justification's `outlist` (`api.py:1393`), and adds the defeated node to `defeater.dependents` (`api.py:1394`).
- `defeater-nodes-carry-defeats-node-and-defeats-justification-metadata` — Defeater nodes store `defeater_type`, `defeats_node`, and `defeats_justification` in metadata and round-trip through `export_network` as ordinary nodes (asserted in `test_defeat_justification_basic` and `test_defeater_in_export`).
- `defeater-scoping-is-per-justification` — A node with two justifications remains `IN` after one is defeated, and only goes `OUT` once all justifications have defeaters (`test_defeat_multiple_justifications`).
- `migrate-retract-to-defeaters-replaces-direct-retraction` — `migrate_retract_to_defeaters` converts an OUT node carrying `retract_reason` into a `migrated-retraction-{id}-j{idx}` defeater node, removes `_retracted`/`retract_reason` metadata, and is reversible by retracting the defeater (`tests/test_migrate_defeaters.py`).
