---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: add_nogood in reasons/network.py

# `Network.add_nogood`

## Purpose

`add_nogood` records a contradiction among a set of nodes and, if that contradiction is currently active, attempts to resolve it by dependency-directed backtracking.

A “nogood” here means: **these nodes must not all be `IN` at the same time**. The function appends a `Nogood` record to `self.nogoods` for audit/traceability, then checks whether every listed node is currently `IN`. If so, it finds a responsible premise and retracts it, which makes at least one contradictory node go `OUT` via normal propagation.

It exists to resolve contradictions automatically while preserving the retracted node in the network, so it can potentially be restored later.

---

## Contract

### Preconditions

- Every ID in `node_ids` must exist in `self.nodes`.
- `node_ids` should be a non-empty list of distinct node ID strings.
- `self.nodes[nid].truth_value` is expected to be `"IN"` or `"OUT"`.

The function does **not** enforce non-empty or distinct input. An empty list is a bug case: `all([])` is `True`, `find_culprits([])` returns `[]`, and the fallback path then indexes an empty candidate list, raising `IndexError`.

### Postconditions

- A `Nogood` object is appended to `self.nogoods` with a unique ID of the form:
  - `nogood-<sorted-node-ids-joined-by-hyphens>`
  - or `...-2`, `...-3`, etc. on collision.
- A `"nogood"` log entry is added to `self.log`.
- If **not all** listed nodes are `IN`, the function returns `[]` and performs no retraction. The nogood is still recorded.
- If all listed nodes are `IN`:
  - If `find_culprits` returns candidates, the first candidate premise is retracted.
  - Otherwise, the fallback retracts the listed node with the fewest dependents.
  - The return value is the result of `self.retract(victim_id)`.
- The function does **not** integrate the nogood into later truth computation. `_compute_truth` and `_justification_valid` never consult `self.nogoods`.

### Important invariant / docstring mismatch

The docstring and inline comment say the function retracts “the premise with the fewest dependents.” The actual implementation retracts `culprits[0]["premise"]`, where `find_culprits` sorts candidates by `_entrenchment` ascending. `_entrenchment` includes dependent count, but also premise/evidence status, source backing, source hash, and beliefs type. It is **not** simply “fewest dependents.”

---

## Parameters

### `node_ids: list[str]`

A list of node IDs that form the contradictory combination. Each ID must be a key in `self.nodes`.

Edge cases:

- Missing node: raises `KeyError`.
- Empty list: reaches fallback with no candidates and raises `IndexError`.
- Duplicate IDs: not rejected. They affect nogood slug generation and stored `Nogood.nodes`, though `find_culprits` collapses duplicates through a dict.
- Non-string values: type hint says strings, but only sorting/joining may fail if values are not mutually comparable.
- Generator instead of list: not supported safely; validation would consume it.

---

## Return Value

Returns `list[str]` of all node IDs whose truth value changed as a result of retraction and propagation.

Cases:

- If any listed node is not `IN`: returns `[]`.
- If all listed nodes are `IN` and a valid `IN` victim is retracted: returns a list starting with the victim ID, followed by propagated dependents.
- If an already-`OUT` premise is selected as culprit: `retract` updates metadata and returns `[]`, so `add_nogood` can return `[]` even though the nogood was active.
- The return value does **not** include the nogood ID or indicate whether a nogood was merely recorded.

Callers must inspect `self.nogoods` or `self.log` if they need the nogood ID.

---

## Algorithm

1. **Validate nodes**
   - Iterate over `node_ids`.
   - If any ID is missing from `self.nodes`, raise `KeyError`.

2. **Build a unique nogood ID**
   - Sort `node_ids` and join with `-`:
     ```python
     slug = "-".join(sorted(node_ids))
     nogood_id = f"nogood-{slug}"
     ```
   - If that ID already exists in `self.nogoods`, append `-2`, `-3`, etc. until unique.

3. **Create and record the nogood**
   - Construct `Nogood(id=..., nodes=list(node_ids), discovered=...)`.
   - Append it to `self.nogoods`.
   - Log `"nogood"` with the new ID and stringified node list.

4. **Check whether the contradiction is active**
   ```python
   all_in = all(self.nodes[nid].truth_value == "IN" for nid in node_ids)
   ```
   - If false, return `[]` immediately.
   - Note that `all([])` is `True`, so empty input incorrectly looks “active.”

5. **Find responsible premises**
   - Call `self.find_culprits(node_ids)`.
   - `find_culprits` traces assumptions for each `IN` nogood node, collects premises, and returns candidates sorted by ascending entrenchment.

6. **Choose the victim**
   - If `culprits` is non-empty:
     ```python
     victim_id = culprits[0]["premise"]
     ```
     This is the least-entrenched candidate, not necessarily the one with fewest direct dependents.
   - Otherwise, fallback:
     ```python
     candidates = [(nid, len(self.nodes[nid].dependents)) for nid in node_ids]
     candidates.sort(key=lambda x: x[1])
     victim_id = candidates[0][0]
     ```
     This picks the nogood node with the fewest dependents. If `node_ids` is empty, this raises `IndexError`.

7. **Retract the victim**
   - Return `self.retract(victim_id)`.
   - `retract` marks the victim `OUT`, sets `_retracted`, timestamps, logs, and propagates truth-value changes to dependents.

---

## Side Effects

- Appends to `self.nogoods`.
- Appends to `self.log`:
  - `"nogood"` always.
  - `"backtrack"` only when `culprits` is non-empty.
  - `"retract"` and propagation logs via `self.retract`.
- Mutates the selected victim node:
  - `truth_value = "OUT"`
  - `metadata["_retracted"] = True`
  - `retracted_at` and `updated_at` timestamps.
- May mutate dependent nodes’ truth values through `_propagate`.
- Does not modify justifications or the dependents index directly.
- `Nogood.discovered` uses `datetime.now().isoformat(timespec="seconds")`, i.e. local naive time. This differs from several other timestamps in the file that use `datetime.now(timezone.utc)`.

---

## Error Handling

- Raises `KeyError` if any `node_ids` entry is not in `self.nodes`.
- Can raise `TypeError` during `sorted(node_ids)` if IDs are not comparable strings.
- Raises `IndexError` for empty `node_ids` in the fallback path.
- Does not validate duplicate IDs.
- Does not catch exceptions from `find_culprits` or `retract`.
- Can silently return `[]` while leaving an active contradiction unresolved if the selected culprit is already `OUT`.

---

## Usage Patterns

Typical call:

```python
changed = network.add_nogood(["belief-a", "belief-b"])
```

Caller obligations:

- Do not assume the function always retracts something. It records the nogood even when the contradiction is inactive.
- Do not assume an empty return means nothing happened; it may mean the nogood was recorded but inactive, or that the selected culprit was already `OUT`.
- Do not expect automatic future enforcement. If a nogood is inactive when recorded, later changes that make all nodes `IN` will not automatically re-trigger resolution. The caller must call `add_nogood` again or resolve it manually.
- Do not rely on “fewest dependents” selection despite the docstring. Inspect `find_culprits` or choose a victim manually if exact retraction policy matters.
- Repeated calls with the same `node_ids` add suffixed nogoods rather than deduplicating.

---

## Dependencies

- Standard library: `datetime`.
- Package imports: `Nogood`.
- Internal methods:
  - `Network.find_culprits`
  - `Network.retract`
  - `Network._log`
  - Indirectly: `trace_assumptions`, `_entrenchment`, `_propagate`, `_compute_truth`, `_justification_valid`.

---

## Assumptions Not Enforced by Type System

- `node_ids` is non-empty and distinct.
- Every `truth_value` is exactly `"IN"` or `"OUT"`.
- `find_culprits` returns a list sorted by ascending entrenchment and that `culprits[0]` is actionable.
- `culprits[0]["premise"]` is currently `IN`. This is not guaranteed: `trace_assumptions` walks all justifications, including invalid ones, and `find_culprits` does not filter premises by truth value. So an already-`OUT` premise can be selected.
- `self.nogoods` elements expose `.id`.
- No concurrent calls mutate `self.nogoods` / `self.nodes` during ID generation.
- Nogoods are not enforced during propagation; they are only resolved at the moment `add_nogood` is called.
- `trace_assumptions` follows only justification antecedents, not `outlist` nodes. A contradiction involving an outlist condition may not identify the relevant absent/`OUT` node as a culprit.

---

## Topics to Explore

- [function] `reasons/network.py:find_culprits` — Explains how premises are traced and ranked by entrenchment; `add_nogood` depends on its ordering and return shape.
- [function] `reasons/network.py:retract` — Shows how the victim is marked `OUT` and how truth-value changes cascade through dependents.
- [function] `reasons/network.py:trace_assumptions` — Details which premises are considered responsible; note it walks all justifications and ignores outlist dependencies.
- [file] `tests/test_backtracking.py` — Likely exercises dependency-directed backtracking and expected victim selection.
- [general] `nogood-enforcement` — Investigate whether nogoods are ever re-checked during propagation, or only at `add_nogood` call time.

---

## Beliefs

- `add-nogood-records-before-active-check` — It appends a `Nogood` and logs it before determining whether all listed nodes are `IN`.
- `inactive-nogood-returns-empty` — If any listed node is not `IN`, `add_nogood` returns `[]` and performs no retraction.
- `active-nogood-selects-least-entrenched-culprit` — When `find_culprits` returns candidates, `add_nogood` retracts `culprits[0]["premise"]`, which is sorted by least entrenchment, not by fewest dependents.
- `nogood-is-not-enforced-during-propagation` — `_compute_truth` and `_justification_valid` never consult `self.nogoods`, so later truth changes do not automatically re-check a recorded nogood.
- `may-retract-already-out-premise` — Because `find_culprits` traces all justifications and does not filter premises by truth value, `add_nogood` can select an already-`OUT` premise and return `[]` without resolving the contradiction.
