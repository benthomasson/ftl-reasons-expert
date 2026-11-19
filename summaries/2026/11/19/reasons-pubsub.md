---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/pubsub.py

# `reasons/pubsub.py`

## Purpose

This module implements a small **in-process publish/subscribe bus for TMS truth-value changes**. Its job is to let other parts of the process observe changes after a write transaction has completed, without coupling the core transaction/write logic to any particular consumer.

The module’s docstring makes the core contract explicit:

- Subscribers receive **batched** `ChangeEvent` objects.
- Events are published after `_with_network`’s write transaction completes, i.e. **after save**.
- Intermediate cascade states are **not** published; only the final stable state is visible.
- Events describe **what changed** (`old_truth_value` / `new_truth_value`) but not **why** it changed. Cause/cascade metadata is intentionally absent because the `_with_network` hook only has before/after snapshots.

This is a module-level singleton bus. It is not a network pub/sub system, not persisted, and not cross-process.

## Key Components

### `ChangeEvent`

```python
@dataclass(frozen=True)
class ChangeEvent:
    type: str
    node_id: str
    old_truth_value: str | None
    new_truth_value: str | None
    db_path: str
```

A frozen dataclass representing one truth-value change.

- `type`: event category. `compute_changes()` currently always emits `"node_changed"`.
- `node_id`: the TMS node whose truth value changed.
- `old_truth_value`: previous truth value, or `None` if the node was not present before.
- `new_truth_value`: new truth value, or `None` when the node disappeared from the `after` snapshot.
- `db_path`: database scope for the event.

Because it is frozen, events are intended to be immutable records passed to subscribers.

### `Subscriber`

```python
Subscriber = Callable[[list[ChangeEvent]], None]
```

A subscriber is any callable that accepts a batch/list of `ChangeEvent`s and returns `None`.

### `_subscribers`

```python
_subscribers: dict[str | None, list[Subscriber]] = {}
```

Module-level registry.

- Key `None`: global subscribers, receiving events from all database paths.
- Key `"some/db/path"`: subscribers scoped to that exact `db_path`.

A given callback can be registered more than once; there is no deduplication.

### `subscribe(callback, db_path=None)`

Registers a subscriber:

```python
_subscribers.setdefault(db_path, []).append(callback)
```

- `db_path=None` registers a global subscriber.
- A specific `db_path` registers a scoped subscriber.
- No validation or duplicate detection.

### `unsubscribe(callback, db_path=None)`

Removes a previously registered subscriber:

```python
subs = _subscribers.get(db_path, [])
try:
    subs.remove(callback)
except ValueError:
    pass
if not subs:
    _subscribers.pop(db_path, None)
```

Important details:

- Uses `list.remove`, so it removes the first matching registration only. If the same callback was registered twice, one unsubscribe removes one registration.
- Unsubscribing a callback that is not registered is a no-op.
- The `db_path` key is removed when the subscriber list becomes empty.

### `has_subscribers(db_path=None)`

Returns whether there are any relevant subscribers.

```python
if _subscribers.get(None):
    return True
if db_path is not None and _subscribers.get(db_path):
    return True
return False
```

Semantics:

- If there is at least one global subscriber, it returns `True`.
- If `db_path` is provided and there is at least one subscriber for that exact path, it returns `True`.
- If `db_path=None`, it only checks global subscribers. Other path-scoped subscribers do not count.

### `publish(events, db_path)`

Dispatches a batch of events to relevant subscribers.

```python
targets = list(_subscribers.get(None, []))
targets.extend(_subscribers.get(db_path, []))
```

- No-op if `events` is empty.
- Targets are the global subscribers plus subscribers registered for the exact `db_path`.
- The target list is copied before iteration, so subscriber additions/removals during dispatch do not mutate the list being iterated.
- Each subscriber is called with the same `events` list.
- Exceptions are caught per subscriber and logged. Other subscribers still run.

### `compute_changes(before, after, db_path)`

Diffs two truth-value snapshots and returns `list[ChangeEvent]`.

Snapshots are `dict[str, str]` mappings from `node_id` to truth value.

Algorithm:

1. For each node in `after`:
   - Compare `before.get(node_id)` with `after[node_id]`.
   - If different, emit a `"node_changed"` event.
2. For each node in `before` that is absent from `after`:
   - Emit a `"node_changed"` event with `new_truth_value=None`.

So:

- Additions produce events with `old_truth_value=None`.
- Truth-value transitions produce events with both old and new values.
- Deletions from the snapshot produce events with `new_truth_value=None`.

The order is deterministic relative to input dict iteration order: changed/added nodes in `after` order first, then removed nodes in `before` order.

### `clear()`

Removes all subscribers:

```python
_subscribers.clear()
```

Intended for test teardown.

## Patterns

### Observer / Pub-Sub

This is a classic in-process observer pattern:

- `_subscribers` is the registry.
- `subscribe` / `unsubscribe` manage observers.
- `publish` notifies them.
- `ChangeEvent` is the payload.

It is intentionally simple and synchronous.

### Scoped subscription keys

The registry uses `db_path` as a lightweight scope mechanism:

- `None` = wildcard/global.
- exact path = scoped.

This avoids needing a more complex routing system while allowing tests or multi-database processes to separate streams.

### Snapshot diffing

`compute_changes()` separates change detection from event delivery. The caller likely captures a `before` snapshot, performs mutations, captures an `after` snapshot, then diffs them. This keeps the pubsub module independent of TMS internals.

### Immutable event DTO

`ChangeEvent` is frozen, which makes it safe to pass the same event objects to multiple subscribers without expecting mutation.

### Failure isolation

`publish()` catches subscriber exceptions and logs them, so a broken subscriber cannot break the write transaction or prevent other subscribers from receiving the batch.

## Dependencies

Imports:

- `logging` for subscriber exception reporting.
- `dataclasses.dataclass` for `ChangeEvent`.
- `typing.Callable` for the `Subscriber` type alias.

No third-party dependencies.

Imported by / used by:

- `reasons/api.py` — likely the main integration point; `_with_network` should call `compute_changes()` and `publish()` after save.
- `tests/test_pubsub.py` — unit tests for subscription, publishing, diffing, and error isolation.
- `reasons_lib/pubsub.py` — the repository contains a mirrored `reasons_lib` package; this module appears duplicated there. A maintainer should verify the copies stay synchronized.

## Flow

A typical write transaction flow is:

1. A caller registers a subscriber:
   - `subscribe(callback)` for all databases, or
   - `subscribe(callback, db_path="/path/to/db")` for one database.

2. During a write transaction, `_with_network` captures the truth-value state before mutation.

3. The transaction performs its writes and saves.

4. After save, the caller captures the stable `after` truth-value state.

5. `compute_changes(before, after, db_path)` builds a list of `ChangeEvent`s.

6. `publish(events, db_path)`:
   - returns immediately if there are no events;
   - collects global subscribers and exact-`db_path` subscribers;
   - invokes each subscriber with the batch;
   - logs and swallows any subscriber exception.

7. Each subscriber receives the final batched change list, not intermediate cascade states.

## Invariants

- Events are published only after a write transaction completes and has saved. The module itself does not enforce this; the caller / `_with_network` integration does.
- Events represent final stable truth-value changes, not intermediate cascade transitions.
- `ChangeEvent` does not contain cause or `cascade_root` metadata.
- `compute_changes()` represents deletion as absence from the `after` snapshot, producing `new_truth_value=None`.
- `compute_changes()` currently only emits `type="node_changed"`.
- Global subscribers (`db_path=None`) receive events for every `db_path` passed to `publish`.
- Path-scoped subscribers receive events only for their exact `db_path`.
- `publish()` is a no-op when `events` is empty.
- Subscriber exceptions do not propagate out of `publish()`.
- The subscriber registry is process-local and mutable; there is no locking, so no thread-safety guarantee is provided.
- Passing `db_path=None` to `publish()` would collect global subscribers twice because both `_subscribers.get(None)` calls match. The signature expects a concrete `str`, so callers should not do that.

## Error Handling

- `unsubscribe()` swallows `ValueError` when the callback is not found. Removing an absent subscriber is safe.
- `publish()` catches `Exception` around each subscriber call and logs with `logger.exception(...)`. It does not retry, remove, or re-raise.
- `compute_changes()` performs no validation and raises only if inputs violate expected mapping behavior.
- `clear()` has no error paths.

## Topics to Explore

- [file] `reasons/api.py` — See where `_with_network` snapshots truth values, calls `compute_changes`, and invokes `publish`.
- [function] `reasons/api.py:_with_network` — Confirm the transaction lifecycle and why intermediate cascade states are suppressed.
- [file] `tests/test_pubsub.py` — Check coverage of scoped subscription, batching, deletion events, and subscriber exception isolation.
- [file] `reasons_lib/pubsub.py` — Compare the mirrored package copy for drift with `reasons/pubsub.py`.
- [general] `tms-cascade-visibility` — Understand the TMS cascade model and why only final stable states are published.

## Beliefs

- `change-event-is-frozen-dataclass` — `ChangeEvent` is an immutable frozen dataclass with fields `type`, `node_id`, `old_truth_value`, `new_truth_value`, and `db_path`.
- `publish-isolates-subscriber-exceptions` — A subscriber exception is caught and logged by `publish()`, and other subscribers still receive the event batch.
- `global-subscribers-receive-db-scoped-events` — A callback registered with `db_path=None` is included for every database path passed to `publish()`.
- `compute-changes-emits-deletions-for-missing-after-keys` — `compute_changes()` emits a `node_changed` event with `new_truth_value=None` when a node exists in `before` but is absent from `after`.
- `publish-noops-on-empty-event-list` — `publish()` returns immediately when `events` is empty and does not notify subscribers.
