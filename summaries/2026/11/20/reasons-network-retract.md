---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: retract in reasons/network.py

## Overview

`Network.retract` is the primary negative-operation on the TMS (Truth Maintenance System) graph: it manually flips a node's truth value to `OUT` and lets the resulting cascade ripple through the dependency graph so that any inference whose support is now broken is itself recomputed. It is the manual counterpart to `assert_node` (which restores) and the shared engine used by automated contradiction resolution (`add_nogood` → backtracking).

---

## 1. Purpose

- **Explicitly falsify/withdraw a belief** without deleting it. Nodes remain in `self.nodes` so that a later `assert_node` can restore them without re-derivation.
- **Trigger recomputation** of every node that transitively depends on the retracted node — both `antecedents` dependents (whose justifications may now be broken) and `outlist` dependents (whose "unless" clause may now be satisfied, potentially reviving them).
- **Mark the node as "sticky OUT"** via `metadata["_retracted"] = True`, which prevents subsequent propagation from silently flipping the node back to `IN`. Only `assert_node` clears this flag.

## 2. Contract

**Preconditions**
- `node_id` **must** be a key in `self.nodes`; otherwise `KeyError` is raised before any mutation.
- `node.truth_value` is expected to be `"IN"` or `"OUT"` (implicit assumption — not enforced by types).
- `node.metadata` is expected to be a mutable dict (guaranteed by `Node` construction in `add_node`, but not asserted here).

**Postconditions**
- `node.truth_value == "OUT"`.
- `node.metadata["_retracted"] is True`.
- `node.retracted_at` and `node.updated_at` equal a fresh UTC ISO-8601 second-precision timestamp.
- If `reason` is truthy, `node.metadata["retract_reason"] == reason`; otherwise any prior `retract_reason` is **left untouched**.
- Every node in the returned list has `truth_value` possibly changed; the list is `[node_id] + <propagation results>` when the node was `IN`, and `[]` when it was already `OUT`.
- `self.log` gains a `"retract"` entry **only** when the node was `IN`.

**Invariants preserved**
- `self.nodes` is never mutated structurally (no add/remove).
- Justifications are untouched — retraction is *not* the same as removing justifications; the node could be re-derived if `_retracted` were cleared.
- The `dependents` reverse index is not modified (dependency structure is orthogonal to truth value).

**Important asymmetry:** `retract` bypasses `_compute_truth`. It forces `OUT` even for premises (which `_compute_truth` normally leaves at their current value). This is intentional — manual retraction is an authoritative override.

## 3. Parameters

| Name | Type | Semantics | Edge cases |
|------|------|-----------|------------|
| `node_id` | `str` | ID of an existing node to retract. | Missing → `KeyError`. Not stripped/normalized; must match exactly. |
| `reason` | `str` (default `""`) | Human-readable rationale stored in `metadata["retract_reason"]`. | Empty string is falsy → **not** stored and does **not** overwrite a previous reason. Non-string values are not validated and would be stored as-is (type assumption). |

## 4. Return Value

A `list[str]` of node IDs whose truth value *actually changed*:

- If the node was already `OUT`: `[]` (but metadata/timestamps are still refreshed — see side effects).
- Otherwise: `[node_id, *dependents_recomputed]`, where `dependents_recomputed` comes from `_propagate`. The retracted node always leads the list; propagation never re-adds it because `_propagate` seeds `visited` with `changed_id`.

**Caller obligations:** treat the list as the set of invalidated/refreshed nodes (e.g., to emit events, invalidate caches, log to a journal). Do not rely on ordering beyond "the retracted node is first." Do not assume the list is non-empty.

## 5. Algorithm

1. **Existence check** — `if node_id not in self.nodes: raise KeyError(...)`. Bail before touching state.
2. **Capture `node` and current UTC time** (`datetime.now(timezone.utc).isoformat(timespec="seconds")`) once for reuse.
3. **Fast path if already `OUT`:**
   - Set `metadata["_retracted"] = True` (sticky flag).
   - Refresh `retracted_at` and `updated_at` to `now`.
   - Return `[]` — **no log entry, no propagation**. This is important: the truth value did not change, so nothing downstream needs recomputation, *but* the node is now pinned OUT against future propagation.
4. **Normal path (node was `IN`):**
   - `node.truth_value = "OUT"`.
   - Set `_retracted`, `retracted_at`, `updated_at`.
   - Conditionally store `retract_reason` only if `reason` is non-empty.
   - Seed `changed = [node_id]`.
   - `self._log("retract", node_id, reason or "OUT")` — the log value is the reason if provided, otherwise the literal `"OUT"`.
   - `changed.extend(self._propagate(node_id))`.
5. **`_propagate(node_id)`** does a BFS over `current.dependents`:
   - Skips any dependent whose `metadata["_retracted"]` is `True` (this is how the sticky flag propagates — a retracted node is never auto-revived).
   - Skips dangling `dep_id` not in `self.nodes`, logging a warning.
   - Recomputes via `_compute_truth`; if the value flips, records it, logs `"propagate"`, and enqueues it.
   - The BFS fixpoint ensures transitive dependents are recomputed in topological-ish order driven by actual change.
6. Return `changed`.

## 6. Side Effects

- **Target node mutations**: `truth_value`, `metadata["_retracted"]`, `retracted_at`, `updated_at`, and optionally `metadata["retract_reason"]`.
- **`self.log`**: appended to via `_log("retract", ...)` (on the IN→OUT transition) and via `_log("propagate", ...)` / `_log("warn", ...)` inside `_propagate`.
- **Downstream node mutations**: `_propagate` calls `_compute_truth` on each reachable dependent, which mutates `node.supporting_justification` (selects the last valid justification index, or `None`). It then mutates `truth_value` on any dependent that flipped.
- **No I/O, no network, no file system access.** All effects are in-memory.
- **No structural mutation** of `nodes`, `nogoods`, or `dependents`.

## 7. Error Handling

- **`KeyError`** on unknown `node_id`. This is the only error intentionally raised.
- No defensive checks on `reason` type — a non-string is stored verbatim.
- No check that `node.metadata` is a dict; a corrupted node would raise `TypeError`/`AttributeError`.
- Dangling dependents encountered during `_propagate` are **swallowed** — logged as `"warn"` events but not surfaced to the caller.
- If a dependent was already `_retracted`, it is silently skipped (this is a feature, not an error).

## 8. Usage Patterns

- **Manual withdrawal** in the CLI / API when a user decides an assumption is no longer valid: `changed = net.retract("assumption-42", reason="source discredited")`.
- **Automated backtracking** inside `add_nogood`: after `find_culprits`, the least-entrenched premise is retracted to break a contradiction. The changed list flows back to the caller as the "what got invalidated" audit.
- **Interactive repair / dialogue**: agents may retract a challenge target and later `assert_node` to reverse.
- **Caller obligations:**
  - Persist the returned `changed` list if downstream state (caches, exports, wiki builds) must be re-synced.
  - If you want a node to become `IN` again after retraction, you **must** call `assert_node`; propagation will *not* restore it because `_retracted` blocks recomputation.
  - Do not rely on `retract` to delete edges or justifications — use `remove_justification` / `convert_to_premise` for that.

## 9. Dependencies

- **Standard library**: `datetime.datetime`, `datetime.timezone`, `collections.deque` (via `_propagate`).
- **Internal**:
  - `Network._propagate` — BFS recomputation.
  - `Network._compute_truth` — truth evaluation used by propagation.
  - `Network._justification_valid` — validates SL/CP justifications during recomputation.
  - `Network._log` — audit trail.
  - `Node` attributes: `truth_value`, `metadata`, `retracted_at`, `updated_at`, `dependents`, `justifications`, `supporting_justification`.
- **Not** dependent on Merit/Merkle hashing, storage, or the CLI — pure graph logic.

## 10. Assumptions Not Enforced by Types

- `node.truth_value ∈ {"IN", "OUT"}` — any other string silently escapes the `== "OUT"` check and proceeds to the normal path.
- `node.metadata` is a mutable mapping; `node.metadata["access_tags"]` etc. written elsewhere assume the same.
- **`dependents` is kept fresh** by callers of `add_node` / `add_justification` / `_rebuild_dependents`. If the reverse index is stale, the cascade will miss dependents silently (no warning for *missing* dependents — only for dangling IDs).
- `reason` is a plain string; no whitespace trimming, no length cap, no sanitization for downstream serializers.
- Timestamps assume the system clock is UTC-consistent; the code stores naive-looking ISO strings (they include `+00:00`).
- Multiple retractions of the same node are assumed to be idempotent-ish; the code refreshes timestamps but **does not** overwrite an existing `retract_reason` if the new call omits one.
- Nodes retracted while already `OUT` (e.g., via invalid justifications) acquire the `_retracted` sticky flag even though nothing propagated — a subtle behavior that can surprise callers expecting "retract on an already-OUT node is a no-op."

---

## Topics to Explore

- [function] `reasons/network.py:_propagate` — The BFS core; understand why `_retracted` dependents are skipped and how fixpoints are reached
- [function] `reasons/network.py:add_nogood` — The automated backtracking path that selects a victim via `find_culprits` and calls `retract`
- [function] `reasons/network.py:assert_node` — The inverse operation; note it clears `_retracted` and `retract_reason` is not cleared automatically
- [file] `tests/test_backtracking.py` — Concrete tests exercising retract-as-contradiction-resolution
- [file] `tests/test_network.py` — Base contract tests for the retract/assert round-trip and idempotency

## Beliefs

- `retract-returns-empty-when-already-out` — If `node.truth_value == "OUT"` on entry, `retract` refreshes `retracted_at`/`updated_at` and sets `_retracted=True` but returns `[]` without logging or propagating.
- `retract-sets-sticky-retracted-flag` — Every call sets `node.metadata["_retracted"] = True`, which causes `_propagate` to skip that node forever until `assert_node` clears it.
- `retract-does-not-overwrite-reason-when-blank` — A falsy `reason` argument leaves any pre-existing `metadata["retract_reason"]` intact.
- `retract-bypasses-compute-truth` — Unlike `_propagate`, `retract` assigns `"OUT"` directly, overriding premises that `_compute_truth` would otherwise leave at their current value.
- `retract-propagates-only-on-in-to-out-transition` — The cascade through dependents only runs when the target transitions from `IN` to `OUT`; the already-OUT branch performs no propagation.
