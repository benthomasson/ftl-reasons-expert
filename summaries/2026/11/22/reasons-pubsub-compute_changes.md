---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: compute_changes in reasons/pubsub.py

## `compute_changes`

**Location:** `reasons/pubsub.py`

### Purpose

`compute_changes` diffs two truth-value snapshots keyed by node id and converts the differences into a list of `ChangeEvent` objects.

It exists so the pub/sub layer can batch truth-value changes after a write transaction. The caller provides the state before and after the transaction, and this function produces the event payload. It does **not** publish events itself; `publish` does that separately.

Because it only sees snapshots, it cannot populate cause/cascade-root metadata; `ChangeEvent` does not include those fields by design.

### Contract

**Preconditions**

- `before` and `after` are mappings from node id to truth value.
- Missing nodes are represented by the key being absent from the dictionary.
- Truth values are expected to be strings; `None` is used internally as the event-level representation of “no value” for a missing side.
- Both snapshots should represent the same database/transaction context, although this is not enforced.
- `db_path` identifies the database associated with the snapshots.

**Postconditions**

- The input dictionaries are not mutated.
- A new list of `ChangeEvent` instances is returned.
- For each node present in `after`:
  - If `before` lacks the node, an event is created with `old_truth_value=None`.
  - If the node exists in both but the truth value differs, an event is created with both old and new values.
  - If the node exists in both with the same truth value, no event is created.
- For each node present in `before` but absent from `after`, an event is created with `new_truth_value=None`.
- Every returned event has `type="node_changed"`, even for additions and deletions.
- Every returned event gets the provided `db_path`.

**Invariants**

- Under normal valid input, at most one event is emitted per node id.
- Event order is:
  1. Nodes encountered while iterating `after.items()`.
  2. Deleted nodes encountered while iterating over `before`.
- The function has no global side effects and does not interact with the subscriber registry.

### Parameters

- `before: dict[str, str]`
  - Snapshot of node truth values before the transaction.
  - Empty dictionaries are allowed.
  - The function uses `before.get(nid)`, so a missing key is treated as `None`.

- `after: dict[str, str]`
  - Snapshot of node truth values after the transaction.
  - Empty dictionaries are allowed.
  - Keys present only in `after` are treated as new nodes, with no previous truth value.

- `db_path: str`
  - Identifies the database associated with the snapshots.
  - It is only copied into every generated `ChangeEvent`.
  - No validation, normalization, or filesystem access is performed.

### Return Value

Returns `list[ChangeEvent]`.

- Returns an empty list when `before` and `after` contain the same node ids and truth values.
- Returns one `ChangeEvent` per added, modified, or deleted node under valid string-valued snapshots.
- The caller must handle an empty list if desired; this function does not special-case or suppress empty results.
- The returned list is newly allocated and can be modified by the caller, though the `ChangeEvent` objects themselves are frozen dataclasses.

### Algorithm

1. Initialize an empty `events` list.
2. Iterate over `after.items()`:
   - Look up the same node id in `before` using `before.get(nid)`.
   - If the old value is different from the new value, append a `ChangeEvent` with:
     - `type="node_changed"`
     - `node_id=nid`
     - `old_truth_value=old_tv`
     - `new_truth_value=new_tv`
     - `db_path=db_path`
3. Iterate over the keys in `before`:
   - If the node id is not present in `after`, append a deletion-style `ChangeEvent` with:
     - `type="node_changed"`
     - `node_id=nid`
     - `old_truth_value=before[nid]`
     - `new_truth_value=None`
     - `db_path=db_path`
4. Return the accumulated list.

This means additions and modifications are discovered in the `after` loop, while deletions are discovered in the second `before` loop.

### Side Effects

None.

- Does not mutate `before`.
- Does not mutate `after`.
- Does not mutate `_subscribers`.
- Does not call `publish`.
- Does not perform I/O.
- Does not log.
- Only allocates a local list and `ChangeEvent` instances.

### Error Handling

`compute_changes` has no explicit error handling and no `try`/`except`.

Under normal valid inputs (`dict[str, str]`), it should not raise. If malformed inputs are provided, exceptions can propagate to the caller:

- `before` or `after` being `None` or lacking mapping methods can raise `AttributeError`.
- Custom truth-value objects whose equality comparison raises can propagate that exception.
- Invalid `db_path` values are not rejected; they are simply stored in events.
- Runtime `None` truth values are not rejected and can produce ambiguous results.

This differs from `publish`, which catches subscriber exceptions and logs them. `compute_changes` does not swallow or report errors internally.

### Usage Patterns

This function is typically used by the `_with_network` write hook after a transaction commits/saves:

1. Capture `before` truth values.
2. Apply writes.
3. Capture `after` truth values.
4. Call `compute_changes(before, after, db_path)`.
5. Pass the resulting events to `publish(events, db_path)` if dispatching is desired.

Caller obligations:

- Call it with final stable snapshots, not intermediate cascade states.
- Use the same logical `db_path` for both event generation and publication routing.
- Do not rely on cause/cascade metadata; it is not available here.
- Treat additions and deletions as represented by `old_truth_value=None` or `new_truth_value=None`, not by distinct event types.

### Dependencies

- `ChangeEvent` from the same module.
- Python built-in `dict` behavior: `.items()`, `.get()`, `in`, and indexing.
- Python equality comparison via `!=`.

The module imports `logging`, `dataclass`, and `Callable`, but `compute_changes` itself only depends directly on `ChangeEvent`.

### Assumptions Not Enforced by the Type System

- `None` is not a valid truth value; absence is meant to be encoded by missing keys.
- `before.get(nid)` uses `None` as the “missing” sentinel, so a present `None` value is indistinguishable from a missing node.
- `before` and `after` correspond to the same database and node namespace.
- Snapshots represent a completed write transaction’s final stable state.
- Truth values are meaningfully comparable with `!=`.
- Event ordering follows dictionary insertion order; callers should not depend on it unless explicitly documented.
- The `db_path` passed into `compute_changes` agrees with the later `publish` routing key.

## Topics to Explore

- [file] `reasons/pubsub.py` — Understand how `compute_changes` fits alongside `subscribe`, `unsubscribe`, and `publish`.
- [function] `reasons/pubsub.py:publish` — See how events are routed to global and per-database subscribers, and how subscriber exceptions are handled.
- [file] `tests/test_pubsub.py` — Check expected behavior for added, changed, deleted, and unchanged nodes.
- [general] `_with_network transaction hook` — Explore where before/after snapshots are captured and when `compute_changes` is called.
- [general] `truth-value snapshot representation` — Clarify whether missing values, deletions, and `None` are distinguished elsewhere in the system.

## Beliefs

- `compute-changes-produces-node-changed-events` — For valid string-valued snapshots, it returns one `"node_changed"` event per added, modified, or deleted node id.
- `missing-before-key-yields-none-old-truth-value` — Keys present only in `after` get `old_truth_value=None`.
- `missing-after-key-yields-none-new-truth-value` — Keys present only in `before` get `new_truth_value=None`.
- `compute-changes-emits-after-order-then-before-only-order` — Events for `after` keys are emitted first in `after` iteration order, followed by deleted `before` keys in `before` iteration order.
- `compute-changes-does-not-publish-or-mutate-state` — It returns a list of events but does not call `publish`, mutate inputs, or modify the subscriber registry.
