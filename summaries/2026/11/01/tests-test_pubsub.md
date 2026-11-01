---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_pubsub.py

# `tests/test_pubsub.py`

## Purpose

This file is the test suite for the **publish-subscribe (pubsub) event layer** in the `ftl-reasons` project. The project models a directed network of logical beliefs (nodes with truth values `IN`/`OUT`), and this test file verifies that the event notification system correctly:

1. Detects truth-value changes between two snapshots of the network.
2. Delivers those change events to registered subscribers (globally or scoped to a specific database).
3. Integrates with the high-level API so that every mutation operation (`add_node`, `retract_node`, `assert_node`, `add_justification`, `challenge`, `defend`, `add_nogood`) emits the expected events.

It owns the responsibility of proving that the pubsub contract is honored at every layer — from the low-level `compute_changes` diffing, through the subscriber registry, up to the API surface.

## Key Components

### `ChangeEvent` (imported from `reasons.pubsub`)

A dataclass (or equivalent) representing a single truth-value transition. Its fields, as evidenced by constructor usage throughout the tests:

| Field | Type | Meaning |
|-------|------|---------|
| `type` | `str` | Always `"node_changed"` in current tests |
| `node_id` | `str` | The node whose truth value changed |
| `old_truth_value` | `str \| None` | Previous value (`"IN"`, `"OUT"`, or `None` for new nodes) |
| `new_truth_value` | `str \| None` | New value (`"IN"`, `"OUT"`, or `None` for deleted nodes) |
| `db_path` | `str` | The database the change pertains to |

### `compute_changes(before, after, db_path) -> list[ChangeEvent]`

Pure diffing function. Takes two dicts mapping `node_id → truth_value`, plus a db path, and returns the minimal list of `ChangeEvent`s describing the transition. Contract:

- No change → empty list.
- New node (`before` missing, `after` present) → `old_truth_value is None`.
- Deleted node (`before` present, `after` missing) → `new_truth_value is None`.
- Value flip (`IN` → `OUT` or vice versa) → both fields populated.
- Unchanged nodes produce **no** event.

### Subscriber registry functions (from `reasons.pubsub`)

- **`subscribe(callback, db_path=None)`** — Register a callback. If `db_path` is omitted, the subscriber receives events for *all* databases; otherwise only events for that specific db.
- **`unsubscribe(callback)`** — Remove a previously registered callback.
- **`has_subscribers() -> bool`** — Returns `True` if any callback is registered (global or scoped).
- **`publish(events, db_path)`** — Deliver a list of `ChangeEvent`s to all matching subscribers. Skips delivery if `events` is empty.
- **`clear()`** — Remove all subscribers. Used in test teardown.

### `cleanup_subscribers` fixture

```python
@pytest.fixture(autouse=True)
def cleanup_subscribers():
    yield
    clear()
```

Because subscribers are **global mutable state**, this `autouse` fixture guarantees every test starts and ends with a clean registry, preventing cross-test contamination.

## Patterns

- **Publish-Subscribe with scoping**: Subscribers can be global (no `db_path`) or scoped to a specific database. The `publish` call checks the event's `db_path` against the subscriber's scope. This is a multi-tenant pubsub where "tenant" = database file.

- **Layered test organization**: Three classes map to three architectural layers:
  - `TestComputeChanges` — pure function, no I/O, no state.
  - `TestSubscriberRegistry` — in-memory state machine (subscribe/unsubscribe/publish).
  - `TestApiIntegration` — end-to-end: API mutation → snapshot diff → event delivery.

- **`autouse` cleanup fixture**: Standard pattern for global-state tests. The `yield`/`clear()` structure is the pytest idiom for "do this after every test in the module."

- **`tmp_path` for isolated databases**: All API integration tests use pytest's `tmp_path` fixture to create a unique SQLite (or similar) database per test, avoiding shared-state flakiness.

- **Behavioral assertions over implementation**: Tests assert *what* events are emitted (node IDs, old/new values) rather than *how* they are computed. E.g., `test_retract_cascade_emits_events` checks that the cascade reaches `a`, `b`, and `c` without asserting the propagation algorithm.

## Dependencies

### This file depends on:

| Import | Role |
|--------|------|
| `pytest` | Test framework; provides fixtures (`tmp_path`, `autouse`) and assertion rewriting |
| `reasons.api` | High-level mutation API (`add_node`, `retract_node`, `assert_node`, `add_justification`, `propagate`, `challenge`, `defend`, `add_nogood`, `set_metadata`, `show_node`, `init_db`) |
| `reasons.pubsub` | The module under test: `ChangeEvent`, `compute_changes`, `subscribe`, `unsubscribe`, `publish`, `has_subscribers`, `clear` |

### What depends on this file:

- Nothing in production code imports from `tests/`.
- The CI pipeline (`.github/workflows/`) invokes `pytest tests/` which collects this module.
- `tests/conftest.py` may provide shared fixtures consumed here.

## Flow

### 1. Unit-level: `compute_changes`

```
before = {"a": "IN", "b": "OUT"}
after  = {"a": "OUT", "b": "OUT"}
         │
         ▼
compute_changes(before, after, "test.db")
         │
         ▼  diff: "a" changed IN→OUT; "b" unchanged
         │
         ▼
[ChangeEvent("node_changed", "a", "IN", "OUT", "test.db")]
```

The function unions the keys of both dicts, compares values, and emits one event per differing key.

### 2. Registry-level: subscribe → publish → deliver

```
subscribe(cb_global)                    # no db_path → receives all
subscribe(cb_scoped, db_path="my.db")   # only receives "my.db" events

publish([event_for("other.db")], "other.db")
  → cb_global fires, cb_scoped does NOT

publish([event_for("my.db")], "my.db")
  → cb_global fires, cb_scoped fires
```

### 3. API integration: mutation → snapshot → diff → publish

```
api.add_node("p1", "A premise", db_path=db)
  │
  ├─ 1. Snapshot BEFORE state (only if has_subscribers())
  ├─ 2. Perform mutation (write to db)
  ├─ 3. Snapshot AFTER state
  ├─ 4. compute_changes(before, after, db)
  ├─ 5. If events non-empty → publish(events, db)
  │       ├─ For each subscriber:
  │       │   ├─ If global OR subscriber.db_path == db:
  │       │   │   try: subscriber(events)
  │       │   │   except: log, continue
  │       │   └─ else: skip
  │       └─ End
  └─ 6. Return to caller
```

This is validated by `test_events_after_save`: the subscriber calls `api.show_node()` and confirms the new value is already persisted — events fire **after** the write commits.

## Invariants

1. **No event without a truth-value change.** `test_no_changes` and `test_no_event_when_no_change` (metadata set) confirm that operations which don't alter IN/OUT state produce zero events.
2. **Empty publish is a no-op.** `test_publish_empty_list_skipped` — if `events == []`, no subscriber is called.
3. **Subscriber isolation.** A raising subscriber must not prevent delivery to subsequent subscribers (`test_subscriber_exception_logged`).
4. **Events fire post-persistence.** `test_events_after_save` proves the DB is in its new state when the callback runs.
5. **A failed mutation emits no events.** `test_no_events_on_exception` — if the API raises (e.g., `KeyError` on nonexistent node), no events are published. The snapshot/diff/publish sequence is short-circuited.
6. **New nodes have `old_truth_value = None`; deleted nodes have `new_truth_value = None`.** This distinguishes "value flipped" from "node appeared/vanished."
7. **Global and scoped subscribers both fire for a matching db.** `test_global_and_scoped_both_receive` confirms the scoping filter is an OR, not an XOR.

## Error Handling

- **Subscriber exceptions are swallowed and logged.** `test_subscriber_exception_logged` registers a `bad_sub` that raises `RuntimeError("boom")` followed by a `good_sub`. After `publish`, `good_sub` still received its events. The implementation wraps each callback invocation in a `try/except` and logs (presumably via `logging`) rather than propagating.
- **API-level exceptions abort the event pipeline.** `test_no_events_on_exception` shows that if `api.retract_node` raises `KeyError`, the `pytest.raises` block catches it and the test asserts `len(received) == 0`. This means the snapshot-diff-publish logic sits *after* the mutation succeeds, or the mutation's exception propagates before reaching the publish step.
- **No retry or queueing.** Events are fire-and-forget. If a subscriber is slow or errors, there's no back-pressure mechanism — the next `publish` call proceeds independently.

---

## Topics to Explore

- [file] `reasons/pubsub.py` — The implementation under test: how `compute_changes` diffs snapshots, how the subscriber registry stores scoping, and where the `try/except` around callback invocation lives.
- [file] `reasons/api.py` — The mutation surface (`add_node`, `retract_node`, `assert_node`, etc.) and where it calls `has_subscribers()`, takes before/after snapshots, and invokes `publish`. Understanding the call sites clarifies the "events fire after save" invariant.
- [function] `reasons/pubsub.py:publish` — The dispatch loop that filters subscribers by `db_path`, handles exceptions, and short-circuits on empty event lists. This is the most security-relevant path (arbitrary user-supplied callbacks).
- [file] `reasons/storage.py` — The persistence layer that `api` functions call. Understanding how "snapshot" is taken (full-table scan vs. dirty-tracking) affects the performance characteristics of `compute_changes`.
- [general] `cascade-propagation` — How `retract_node("a")` cascades to `b` and `c` (tested in `test_retract_cascade_emits_events`). The pubsub system only reports the *result* of propagation; the traversal logic lives elsewhere.

## Beliefs

- `api-mutations-emit-pubsub-events` — Every truth-value-altering API call (`add_node`, `retract_node`, `assert_node`, `add_justification`, `challenge`, `defend`, `add_nogood`) publishes `ChangeEvent` objects via `reasons.pubsub.publish` *after* the database write succeeds.
- `subscriber-scope-is-database-path` — A subscriber registered with `db_path="X.db"` receives events only when the published batch's `db_path` equals `"X.db"`; global subscribers (no `db_path`) receive all events unconditionally.
- `subscriber-exception-isolation` — An exception raised inside one subscriber's callback is caught and logged; it does not prevent delivery to subsequent subscribers in the same `publish` call.
- `compute-changes-uses-none-for-existence` — `compute_changes` encodes node creation as `old_truth_value=None` and node deletion as `new_truth_value=None`, distinguishing existence transitions from value flips.
- `events-fire-post-persistence` — Subscriber callbacks can call `api.show_node()` and observe the *new* truth value, guaranteeing the event is emitted after the database commit, not before.
