---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: defeat_justification in reasons/api.py

# `defeat_justification` — Explanation

## Purpose

`defeat_justification` is a graph-native way to defeat a single justification of a belief. Instead of marking a node as "retracted" via metadata strings, it creates a real **defeater premise node** in the network and appends that node's ID to the target justification's `outlist`. Because an outlist entry that is `IN` blocks the justification from firing, the target belief naturally goes `OUT` through the normal TMS propagation — and later, if the defeater itself is challenged or retracted, the target can go back `IN`. This is the "first-class defeater" alternative to the older `retract_reason` metadata approach.

It exists to make retraction/defeat **reversible and graph-traversable**, and so that *why* a belief went out is itself a first-class belief with provenance.

## Contract

**Preconditions**
- The database at `db_path` must exist and be loadable by `Storage`.
- `node_id` must exist in `net.nodes` — otherwise `KeyError`.
- The node must have at least one justification — otherwise `ValueError`.
- `justification_index` must be in `[0, len(node.justifications))` — otherwise `IndexError`.

**Postconditions (on successful save)**
- A new premise node with ID `defeater_id` (auto-generated if not injected) exists in the network, with text embedding `reason` and metadata `{defeater_type, defeats_node, defeats_justification, [defeat_reason_type]}`.
- `justification.outlist` contains `defeater_id` (idempotent — only appended if not already present).
- `defeater.dependents` contains `node_id` (reverse index maintained manually).
- `node.updated_at` is refreshed to the current UTC timestamp.
- `net.recompute_all()` has run, so downstream truth values are updated.
- `_with_network(write=True)` persists the DB and, if there are pubsub subscribers, publishes truth-value change events.

**Invariants the code assumes but does not enforce**
- `defeater_id` is unique. There is **no collision check** (unlike `defeat_with_scope`, which de-duplicates scope IDs with a suffix loop but still doesn't guard `defeater_id`).
- The defeater is created as a **premise** (no justifications), so it evaluates `IN` by default — that is what causes the outlist block.
- The outlist entry is what actually triggers the switch; the defeater being `IN` is the graph-level blocker.

## Parameters

| Param | Type | Meaning / Edge cases |
|---|---|---|
| `node_id` | `str` | The belief whose justification is being defeated. Must exist. |
| `justification_index` | `int` | 0-based index into `node.justifications`. Negative or `≥ len` → `IndexError`. |
| `reason` | `str` | Human-readable explanation. **Becomes part of the defeater node's text** (and therefore its `text_hash`). |
| `defeater_type` | `str` | Default `"invalid-inference"`. Stored in metadata and used to build the auto-ID. |
| `defeater_id` | `str \| None` | If `None`, generated as `f"{defeater_type}-{node_id}-j{index}"`. No uniqueness check. |
| `defeat_reason_type` | `str` | Logical failure-mode classification. Only written to metadata if non-empty (empty-string sentinel). |
| `db_path` | `str` | SQLite path; defaults to `DEFAULT_DB = "reasons.db"`. **No `pg_conninfo`** — this function is SQLite-only. |

## Return Value

```python
{
  "node_id": str,
  "justification_index": int,
  "defeater_id": str,
  "defeater_type": str,
  "defeat_reason_type": str,   # echoes input (possibly "")
  "changed": list[str],        # only nodes whose truth_value actually changed
}
```

`changed` is **not** the same as `net.recompute_all()`'s return — the code diffs against a snapshot taken before the defeater was added, so it contains only nodes whose truth value flipped. It may be empty if the justification was already blocked by another outlist entry (i.e., adding the defeater was a no-op).

## Algorithm

1. Enter `_with_network(db_path, write=True)` — loads fresh network, snapshots truth values for pubsub diffing, and saves on clean exit.
2. Validate `node_id` exists (`KeyError` otherwise).
3. Validate `node.justifications` is non-empty (`ValueError` otherwise).
4. Validate `justification_index` in range (`IndexError` otherwise).
5. Grab `justification = node.justifications[justification_index]`.
6. If `defeater_id` is falsy, derive `f"{defeater_type}-{node_id}-j{justification_index}"`.
7. Build `defeater_text` embedding the reason and the target.
8. Snapshot `before = {nid: n.truth_value ...}`.
9. Build `meta` with `defeater_type`, `defeats_node`, `defeats_justification`, and conditionally `defeat_reason_type`.
10. `net.add_node(id=defeater_id, text=defeater_text, metadata=meta)` — creates a premise (no justifications → `IN`).
11. Append `defeater_id` to `justification.outlist` if not already present.
12. **Manually** add `node_id` to `defeater.dependents` — required because `add_node` builds reverse edges only from justifications, and a premise has none.
13. Stamp `node.updated_at`.
14. `net.recompute_all()` — cascades out the target (and its dependents) because the outlist holds an `IN` blocker.
15. Diff `before` vs `after`; return only actually-flipped nodes.

## Side Effects

- **Graph mutation**: adds one node, one outlist edge, one dependents edge.
- **Persistence**: `_with_network` calls `Storage.save` on clean exit.
- **Pub/sub**: if `pubsub.has_subscribers(db_path)`, pre/post truth values are diffed and events published.
- **Cache**: on success, `_network_cache[db_path]` is refreshed to the saved network + current mtime.
- **Timestamps**: `node.updated_at` is overwritten to `now(UTC).isoformat(timespec="seconds")`.
- **Hashing**: since `reason` is embedded in `defeater_text`, the defeater's `text_hash` and any chain hashes depend on `reason` — a caller changing the reason changes the "identity" of the defeater even if `defeater_id` is unchanged.

## Error Handling

- **Raises**:
  - `KeyError(f"Node '{node_id}' not found")`
  - `ValueError(f"Node '{node_id}' has no justifications to defeat")`
  - `IndexError(...)` for out-of-range index.
- **Not caught**:
  - Anything from `net.add_node` (e.g., duplicate-ID behavior decided by `Network`).
  - Anything from `net.recompute_all()` (propagation errors).
  - Storage I/O errors during save.
- On any exception, `_with_network.__exit__` sees `exc_type is not None`, so **the save is skipped** — DB remains consistent with the pre-call state, though in-memory mutations are discarded.

## Usage Patterns

- Typically reached from the CLI or agent code when a review has identified a bad inference and you want the failure to be *first-class and reversible*, not a metadata flag.
- Callers usually follow up with `defeat_with_scope` when they want a defeater whose own justification references "scope beliefs" (so that challenging a scope belief restores the original).
- `migrate_retract_to_defeaters` contains an **inline copy** of this logic (steps 10–14) because it already holds a `_with_network` context — it can't re-enter (nested `_with_network(write=True)` would risk cache/save races).
- Caller obligations: pass a stable `defeater_id` if you plan to reference it later; do not rely on `changed` being non-empty; call from the SQLite backend (no PG dispatch here).

## Dependencies

- `_with_network` (module-local context manager).
- `Network.add_node`, `Network.recompute_all` from `reasons/network.py`.
- `datetime` / `timezone`.
- `Storage`, `pubsub` indirectly via `_with_network`.
- `Justification` is *not* directly used here; it is only imported by sibling functions.

---

## Topics to Explore

- [function] `reasons/api.py:defeat_with_scope` — Builds a derived defeater whose SL justification cites scope beliefs, enabling reversibility through those scope beliefs
- [function] `reasons/api.py:migrate_retract_to_defeaters` — Inline reimplementation of the same defeat logic; highlights the nested-`_with_network` constraint
- [function] `reasons/network.py:recompute_all` — The propagation engine that turns outlist blockers into `OUT` cascades (the mechanism this function relies on)
- [general] `pubsub-change-events` — How `_with_network` diffs pre/post truth values and publishes events; relevant to any write-side side effect
- [file] `tests/test_defeat_justification.py` — Concretely pins down ordering, outlist idempotency, and error-raising behaviors

## Beliefs

- `defeat-justification-requires-in-range-index` — The function raises `IndexError` when `justification_index` is negative or `>= len(node.justifications)`; no clamping occurs.
- `defeat-justification-produces-premise-defeater` — The defeater is added via `net.add_node` with no justifications, so it evaluates `IN` by default and blocks the target's outlist.
- `defeat-justification-manually-updates-dependents` — Because premises have no justifications, the reverse edge `defeater.dependents.add(node_id)` is added explicitly rather than derived by `add_node`.
- `defeat-justification-is-idempotent-on-outlist-but-not-on-node` — Appending `defeater_id` to `outlist` is guarded, but creating the defeater node is not — a repeat call with a colliding `defeater_id` relies on `Network.add_node`'s duplicate semantics.
- `defeat-justification-returns-only-flipped-nodes` — `changed` is computed by diffing against a pre-add snapshot, not from `recompute_all()`'s return, so it can be empty when the justification was already blocked.
- `reason-is-embedded-in-defeater-text` — `reason` is concatenated into the defeater's text, so it participates in `text_hash` and Merkle chain hashing.
