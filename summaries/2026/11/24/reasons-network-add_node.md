---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: add_node in reasons/network.py

# `Network.add_node` — Explanation

**One-liner:** Insert a new `Node` into the `Network`, wire up reverse dependency edges for its justifications, lazily compute Merkle hashes, inherit access-control tags from ancestors, and set the node’s initial `truth_value` (premises default to `IN`; derived nodes are recomputed from their justifications).

---

## 1. Purpose

`add_node` is the **single entry point for inserting a belief-node into the network**. It has to do several things atomically-enough to leave the network usable:

- **Enforce ID uniqueness** — the network uses `id` as the primary key in `self.nodes`.
- **Register reverse edges** in the `dependents` index so truth-change propagation (`_propagate` / BFS) can walk *forward* from antecedents to the nodes they support.
- **Compute provenance hashes** (`text_hash`, per-justification `content_hash`) so the graph can be deduplicated/serialized stably (see `tests/test_deduplication.py`, `tests/test_merkle.py`).
- **Propagate `access_tags`** metadata from ancestor nodes down to this node so downstream filtering by ACL works.
- **Assign an initial truth value**, applying the TMS rule: a premise (no justifications) is `IN`; a derived node is `IN` iff at least one of its justifications is currently valid.
- **Append an audit-trail entry** to `self.log`.

Everything else in the network (retraction, supersession, contradiction resolution) assumes nodes *already exist* and are properly wired. `add_node` is the bootstrap.

---

## 2. Contract

**Preconditions (enforced):**
- `id` is not already in `self.nodes` — otherwise `ValueError`.

**Preconditions (assumed, *not* enforced by the type system):**
- Every `Justification.antecedents` / `.outlist` node ID that should participate in propagation **must already exist** in `self.nodes`. The function silently skips IDs not yet present (see “Assumptions” below).
- `metadata`, if provided non-empty, will be **mutated** later (`access_tags` key added, etc.). If an empty dict is passed, it is *replaced* by a fresh dict.
- `id` should not appear in its own justifications — self-references aren’t handled correctly (the pre-insertion check `ant_id in self.nodes` fails for the node’s own ID, so the self-edge is dropped, and `_compute_truth` then reads the constructor’s default).

**Postconditions (on success):**
- `self.nodes[id]` holds the new `Node`.
- For every justification antecedent/outlist ID that existed at insertion time, `self.nodes[ancestor].dependents` contains `id`.
- `node.text_hash` is populated; each justification has `content_hash` populated (unless already set by the caller).
- `node.metadata["access_tags"]` merged from all antecedent nodes’ `access_tags`.
- `node.truth_value` is `"IN"` or `"OUT"`, never empty.
- One `add` entry is appended to `self.log`.

**Invariants that should still hold afterwards:**
- `self.nodes` IDs are unique and stable.
- The `dependents` index is *superset*-correct for this node: every ancestor already in the network points back at `id`.
- Truth values are internally consistent with justification validity at insertion time (no iterative fixpoint is run — see `recompute_all` for that).

**Non-transactional:** If any step after `self.nodes[id] = node` raises (Merkle import, `_inherit_access_tags`, `_compute_truth`, `_log`), the node is left in `self.nodes` **with partially initialized state** and stale edges. There is no rollback.

---

## 3. Parameters

| Parameter | Type | Meaning | Notes |
|---|---|---|---|
| `id` | `str` | Primary key for the node | Must be unique; used verbatim in logs and dependents index. |
| `text` | `str` | Human-readable claim text | Hashed via `compute_text_hash`; no uniqueness constraint. |
| `justifications` | `list[Justification] \| None` | Support rules | `None` and `[]` both become `[]`; a **non-empty** list is stored by reference, so subsequent mutation by the caller is visible to the network. `type` may be `"SL"` or `"CP"` (both handled identically by `_justification_valid`). |
| `source`, `source_url`, `source_hash`, `date` | `str` | Provenance metadata | Copied into `Node` as-is; no validation. `source` and `source_hash` also affect `_entrenchment`. |
| `metadata` | `dict \| None` | Free-form dict | **Non-empty dict is stored by reference and mutated** (e.g. `access_tags` added). Empty/`None` is replaced with a fresh dict. No schema enforcement. |
| `created_at`, `updated_at` | `str` | ISO 8601 timestamps | If `created_at` empty → both are set to `now` (UTC, seconds precision). If `created_at` provided but `updated_at` not → `updated_at` is **left as `""`** (gap, see below). |
| `reviewed_at`, `verified_at`, `retracted_at` | `str` | Lifecycle timestamps | Copied as-is; `retracted_at` normally set later by `retract`, but can be pre-populated. |

**Interesting edge cases:**
- `justifications=[]` → fresh `[]` used; `justifications=<non-empty>` → caller’s list used directly.
- `metadata={}` → fresh dict; `metadata={"scope": "x"}` → caller’s dict used directly and later mutated.
- `created_at="..."` with `updated_at=""` → `updated_at` ends up empty string. The function only defaults `updated_at` inside the `if not created_at` block.
- Pre-populated `content_hash` on a justification or `text_hash` on the node is **preserved** (the `if not ...` guards). Useful for roundtrip imports (`tests/test_import_json.py`, `tests/test_roundtrip.py`).

---

## 4. Return Value

Returns the newly constructed **`Node` instance** (the same one now stored in `self.nodes[id]`).

Caller can inspect:
- `node.truth_value` — `"IN"` / `"OUT"` as of insertion.
- `node.text_hash` and `j.content_hash` — Merkle roots for dedup / verification.
- `node.metadata["access_tags"]` — inherited tags (present only if some ancestor had tags).

Never returns `None`. Raises rather than returning sentinel values.

---

## 5. Algorithm (step-by-step)

1. **Duplicate guard.** `if id in self.nodes: raise ValueError`.

2. **Timestamp defaults.** If `created_at` is falsy:
   ```
   now = datetime.now(timezone.utc).isoformat(timespec="seconds")
   created_at = updated_at = now
   ```
   (Only `created_at` is checked — `updated_at` isn’t independently defaulted.)

3. **Construct the `Node`.** All string args passed through; `justifications or []` and `metadata or {}` normalize `None`.

4. **Register reverse dependency edges** (for each `j` in `node.justifications`):
   - For each `ant_id in j.antecedents`, if `ant_id in self.nodes`, add `id` to `self.nodes[ant_id].dependents`.
   - Same for each `out_id in j.outlist`.
   - **Order note:** this runs *before* `self.nodes[id] = node`, so a self-reference (`ant_id == id`) is skipped.

5. **Insert.** `self.nodes[id] = node` — the only mutation that makes the node “exist” externally.

6. **Merkle hashes (lazy import).**
   - `from .merkle import compute_text_hash, compute_content_hash` — imported inside the function to avoid a module-load cycle (`merkle` presumably depends on `Node`/`Network`).
   - If `node.text_hash` is falsy, compute it, then for every justification with empty `content_hash`, compute `compute_content_hash(node, j, self)`. The function expects `self` to be in a consistent state for lookup (node already inserted).

7. **Inherit access tags.** `_inherit_access_tags(node)` merges `ant.metadata["access_tags"]` from every antecedent (not from outlist nodes — semantically: being *defeated by* something doesn’t mean sharing its scope).

8. **Compute initial truth value.**
   - If `node.justifications`: `node.truth_value = self._compute_truth(node)`. That returns `"IN"` if **any** justification’s `_justification_valid(...)` is true, else `"OUT"`. For SL/CP justifications, valid means: all antecedents exist **and** are `IN`, and every outlist node either doesn’t exist or is `OUT`.
   - Else (premise): `"IN"` directly.
   - Consequence: **forward-referenced antecedents make the justification invalid → node becomes `OUT`**, not an error.

9. **Log.** `self._log("add", id, node.truth_value)` — appends `{timestamp, action, target, value}` to `self.log`.

10. **Return** the `Node`.

No propagation is triggered here — dependents of the *new* node are empty by construction, and no existing node’s truth can change as a result of an insertion (assuming antecedents were already present and stable).

---

## 6. Side Effects

Mutations to `self`:
- `self.nodes[id] = node` — adds entry.
- `self.nodes[ant_id].dependents.add(id)` — for existing antecedent / outlist nodes.
- `self.log.append({...})` — one `add` audit entry.
- `node.metadata["access_tags"]` — added if any ancestor had tags (mutates the caller’s `metadata` dict if non-empty).

Mutations to *arguments*:
- **Non-empty `metadata` dict is mutated** (`access_tags` key). Caller must not assume it stays clean.
- **Non-empty `justifications` list is stored by reference** and its `content_hash` fields are set. Caller must not reuse the same list for another node.
- Non-empty `justifications`' `Justification` objects get their `content_hash` mutated.

Mutations to the created node:
- `text_hash`, per-justification `content_hash`, `truth_value`, `supporting_justification` (via `_compute_truth`), `metadata["access_tags"]`.

External / module-level:
- Lazy `from .merkle import ...` — import side effects of `merkle` (typically trivial but conceptually I/O-ish).

---

## 7. Error Handling

**Raised explicitly:**
- `ValueError(f"Node '{id}' already exists")` — the only deliberate exception.

**Propagated from underneath (not caught):**
- `ImportError` from `.merkle` if the module can’t be loaded (shouldn’t happen at runtime; would indicate environment breakage).
- Any exception from `Node(...)` constructor — nothing has been mutated at that point (safe).
- Any exception from `_inherit_access_tags`, `_compute_truth`, `compute_content_hash`, or `_log` — happens *after* the node has been inserted. **Network is left in a partial state**; no compensating delete.

**Silently swallowed / ignored:**
- Nonexistent antecedent/outlist IDs in justifications → the edge registration is skipped and the justification becomes invalid for truth computation. **No warning is logged for insertion-time missing refs** (contrast with `_propagate`, which logs a `warn` for dangling dependents).
- Self-reference in justifications → dependent edge is silently not created; the justification remains in `node.justifications` and may produce a wrong truth value.

---

## 8. Usage Patterns

Typical call sequence:

```python
# Premises first (so dependents can be wired and truth computed correctly)
net.add_node("p1", "R script emits errors on empty df", source="issue #42")
net.add_node("p2", "Issue opened by user X", source_url="https://...")

# Derived node
net.add_node(
    "d1",
    "The bug is reproducible",
    justifications=[Justification(type="SL", antecedents=["p1", "p2"], label="supports")],
)
```

- **Order matters:** adding `d1` before `p1`/`p2` skips their edges and evaluates `d1` to `OUT`. Use `_rebuild_dependents` / `verify_dependents` after any import that violates topological order.
- **Called internally** by `summarize(...)` to create summary nodes with `SL["summarizes"]` justifications — callers using `summarize` don’t call `add_node` directly.
- **Callers should not re-use `metadata` dicts or non-empty `justifications` lists** across multiple `add_node` calls, since they are stored (and mutated) by reference.
- **Roundtrip importers** may precompute `text_hash`/`content_hash` to keep hashes stable across serializations; the function respects pre-populated values via the `if not ...` guards.
- **Post-call sanity check** (recommended): `net.verify_dependents() == []` ensures the dependents index is consistent — especially after bulk imports (see `tests/test_dependents_integrity.py`, `tests/test_dangling_dependents.py`).

---

## 9. Dependencies

**Internal module imports (at call time):**
- `from .merkle import compute_text_hash, compute_content_hash` — lazily imported to avoid a `network` ↔ `merkle` import cycle.

**Package imports (at module load):**
- `Node`, `Justification` from `reasons` (`.`).
- `datetime`, `timezone` from `datetime` (used for `created_at`/`updated_at` defaults and logs).

**Internal methods used:**
- `self._inherit_access_tags(node)` — merges tags from antecedents.
- `self._compute_truth(node)` — consults `_justification_valid` on each justification.
- `self._log(...)` — appends to `self.log`.

**State it reads/writes:**
- `self.nodes` — primary key dict.
- `self.log` — audit trail.

---

## Assumptions the type system does NOT enforce

1. **Antecedent existence order.** `List[Justification]` says nothing about whether IDs resolve. The function silently drops references to not-yet-existing nodes; `_compute_truth` then treats such justifications as invalid. The dependents index becomes permanently incomplete (no re-scan at `add_node` time on the *other* side).
2. **Self-reference disallowed.** Nothing in the signature prevents `id in j.antecedents`, and the failure mode is silent (edge dropped, then truth computed against the `Node` constructor’s default `truth_value` — an uninitialized read).
3. **`updated_at` coupling.** `updated_at` is defaulted only when `created_at` is empty. Passing `created_at="..."` alone yields `updated_at == ""`. Nothing in the type reflects this coupling.
4. **Reference vs copy semantics** of `metadata` and `justifications` are **truthiness-dependent**: empty → fresh copy, non-empty → shared reference. A caller relying on either behavior across cases will be surprised.
5. **No transactional guarantee.** The node can exist in `self.nodes` with an unset `truth_value`/partial hashes if an exception fires after insertion. Callers should treat a caught exception from `add_node` as “network possibly corrupted” and run `verify_dependents` / `recompute_all`.
6. **`justification.type` coverage.** `_justification_valid` only accepts `"SL"` and `"CP"`. Any other type string makes the justification invalid (node → `OUT`), silently, without raising in `add_node`.

---

## Topics to Explore

- [file] `reasons/merkle.py` — Read `compute_text_hash` and `compute_content_hash` to see exactly which fields feed the hashes (and whether outlist nodes contribute, which affects dedup semantics).
- [function] `reasons/network.py:_compute_truth` — The single source of truth for truth evaluation; understanding `_justification_valid` is essential for predicting what `add_node` sets `truth_value` to.
- [function] `reasons/network.py:_rebuild_dependents` — The canonical repair path for out-of-order insertions; contrasts with `add_node`’s incremental dependents wiring.
- [file] `reasons/__init__.py` — Where `Node`, `Justification`, and `Nogood` are defined; check the `Node` constructor default for `truth_value` to reason about self-reference behavior.
- [general] `dangling-dependents` — The forward-reference problem Class (`tests/test_dangling_dependents.py`, `tests/test_dependents_integrity.py`); why `add_node` cannot be the only way to populate the dependents index.

## Beliefs

- `add-node-raises-on-duplicate-id` — `add_node` raises `ValueError` and makes no state changes if `id` is already present in `self.nodes`.
- `add-node-updated-at-not-defaulted-when-created-at-provided` — `updated_at` is defaulted to `now` only inside the `if not created_at` branch, so passing `created_at` without `updated_at` leaves `updated_at == ""`.
- `forward-reference-antecedents-are-silently-ignored-at-add-node` — Justification antecedents/outlist IDs not present in `self.nodes` at call time are skipped during dependents registration and treated as invalid by `_justification_valid`, causing the node to default to `OUT` when it has justifications.
- `add-node-lazy-imports-merkle-to-avoid-circular-dependency` — `compute_text_hash` and `compute_content_hash` are imported inside the function body rather than at module top to avoid a `network` ↔ `merkle` import cycle.
- `add-node-has-no-partial-failure-rollback` — If `_inherit_access_tags`, `_compute_truth`, `compute_content_hash`, or `_log` raise after `self.nodes[id] = node`, the node remains inserted with partially initialized state; callers must run `verify_dependents` / `recompute_all` to repair.
