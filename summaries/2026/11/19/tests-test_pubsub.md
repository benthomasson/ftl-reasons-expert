---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_pubsub.py

# `tests/test_pubsub.py`

## Purpose

This file is the executable contract for the `reasons.pubsub` subsystem and its integration into `reasons.api`. It verifies that truth-value changes are:

1. Computed correctly as `ChangeEvent` objects.
2. Dispatched through a global and/or DB-path-scoped subscriber registry.
3. Emitted by API mutations only when truth values actually change.
4. Published after the database save, so subscribers can safely query updated state.
5. Isolated from subscriber failures and from failed API mutations.

It owns no production behavior. It is a pytest suite that constrains the public behavior of `reasons.pubsub` and the API’s event-emission hooks.

## Key Components

### Autouse cleanup fixture

```python
@pytest.fixture(autouse=True)
def cleanup_subscribers():
    yield
    clear()
```

- Runs for every test in this module.
- Ensures global and db-scoped subscriber registries do not leak between tests.
- `clear()` is expected to remove all subscribers, both global and scoped.

### `ChangeEvent`

Although the class implementation is not in this file, the tests define its observable contract:

- Positional construction is used:
  ```python
  ChangeEvent("node_changed", "a", None, "IN", "db1.db")
  ```
- Field order is effectively:
  1. `type`
  2. `node_id`
  3. `old_truth_value`
  4. `new_truth_value`
  5. `db_path`
- `type` is currently `"node_changed"`.
- Truth values are strings such as `"IN"` and `"OUT"`.
- `None` represents “no prior state” or “deleted state”.
- `isinstance(e, ChangeEvent)` is asserted, so it is a real class, not a plain dict.

### `compute_changes(before, after, db_path)`

A pure diff function tested by `TestComputeChanges`.

Contract:

- `before` and `after` are dict-like mappings from `node_id` to truth-value string.
- Returns a list of `ChangeEvent`.
- Emits an event when:
  - A node exists only in `after`: `old_truth_value=None`, `new_truth_value=<new value>`.
  - A node exists only in `before`: `old_truth_value=<old value>`, `new_truth_value=None`.
  - A node exists in both but the truth value differs.
- Does not emit for unchanged nodes.
- New nodes with value `"OUT"` still produce an event.
- The suite does not constrain ordering for multiple events; `test_multiple_changes` compares as a set.

### Subscriber registry functions

Tested in `TestSubscriberRegistry`:

- `subscribe(callback, db_path=None)`
  - Registers a callback.
  - If `db_path` is omitted/`None`, the callback is global and receives events for all DBs.
  - If `db_path` is given, the callback receives only events published for that exact `db_path`.
- `unsubscribe(callback)`
  - Removes a registered callback.
  - Tested for a global subscriber.
- `has_subscribers()`
  - Returns `True` if any global or db-scoped subscriber exists.
- `clear()`
  - Removes all subscribers.
  - After `clear()`, `has_subscribers()` is `False`.
- `publish(events, db_path)`
  - Dispatches an event list to matching subscribers.
  - Callback signature is `callback(events)`, where `events` is the whole list, not one event at a time.
  - `publish([], "t.db")` must not invoke any subscribers.
  - Global subscribers receive all publishes.
  - Scoped subscribers receive only publishes whose `db_path` matches their subscription.
  - Subscriber exceptions must not prevent later subscribers from receiving the event.

### API integration tests

`TestApiIntegration` verifies these API operations emit pubsub events:

- `api.add_node(...)`
- `api.retract_node(...)`
- `api.assert_node(...)`
- `api.add_justification(...)`
- `api.propagate(...)`
- `api.challenge(...)`
- `api.defend(...)`
- `api.add_nogood(...)`
- `api.set_metadata(...)` should not emit because it does not change truth values.

Specific behaviors:

- Adding a premise emits `old=None`, `new="IN"`.
- Retracting a node cascades to dependent derived nodes; all affected nodes emit `IN -> OUT`.
- Re-asserting or adding a justification can emit `OUT -> IN`.
- `propagate()` on a consistent network may emit nothing, but any emitted events must be well-formed.
- `challenge()` and `defend()` produce events for the challenged/defended target.
- `add_nogood()` causes backtracking and at least one node changes.
- Events are emitted after save: a callback can call `api.show_node(...)` and see the new truth value.
- A failed mutation, such as `retract_node("nonexistent")`, raises `KeyError` and emits no events.

## Patterns

- **Observer / pub-sub pattern**: `subscribe`, `unsubscribe`, `publish`, and `clear` implement callback dispatch.
- **Global vs scoped channels**: subscribers can listen to all DBs or a specific `db_path`.
- **Pure diff + side-effect dispatch**: `compute_changes` calculates events; `publish` handles delivery.
- **Post-commit notification**: API mutations publish after the DB state is saved.
- **Unit + integration split**:
  - Unit tests for pure `compute_changes`.
  - Unit tests for registry mechanics.
  - Integration tests through `reasons.api` with temporary SQLite DBs.
- **Pytest fixture isolation**: autouse fixture clears subscriber state after each test.
- **Closure-based callbacks**: tests use lambdas and nested functions to collect events and verify call counts.
- **Temporary DB isolation**: `tmp_path` ensures each integration test has its own DB file.

## Dependencies

### Imports

- `pytest`
  - Provides test discovery, `pytest.raises`, `tmp_path`, and the autouse fixture mechanism.
- `reasons.api`
  - Production API under integration test.
- `reasons.pubsub`
  - `ChangeEvent`, `clear`, `compute_changes`, `has_subscribers`, `publish`, `subscribe`, `unsubscribe`.

### What depends on this file

- Pytest/CI runs this file as part of the test suite.
- No production code imports this test module.
- The test constrains changes to `reasons/pubsub.py` and the event-emission paths in `reasons/api.py`.
- The repository contains a parallel `reasons_lib` tree, but this file imports `reasons`, not `reasons_lib`.

## Flow

1. Pytest imports the module.
2. For each test, `cleanup_subscribers` starts before the test body.
3. `compute_changes` tests call the pure diff function with plain dicts and assert `ChangeEvent` fields.
4. Registry tests:
   - Register callbacks.
   - Publish event lists.
   - Assert which callbacks were invoked and what events they received.
   - Verify `clear()` and `unsubscribe()` reset registry state.
5. Integration tests:
   - Create an isolated SQLite DB under `tmp_path`.
   - Optionally subscribe callbacks.
   - Call `reasons.api` mutations.
   - Assert received events match the expected truth-value transitions.
   - In one test, query `api.show_node` inside the callback to prove events are emitted after save.
6. After each test, the autouse fixture calls `clear()` to remove all subscribers.

## Invariants

- `compute_changes` emits `ChangeEvent(type="node_changed", ...)` for additions, deletions, and value changes only.
- New nodes use `old_truth_value=None`.
- Deleted nodes use `new_truth_value=None`.
- `publish([])` must not call subscribers.
- A global subscriber receives events regardless of `db_path`.
- A scoped subscriber receives events only when `publish(...)` is called with the exact matching `db_path`.
- `has_subscribers()` is true if any global or scoped subscriber exists.
- `clear()` removes all subscribers.
- Subscriber callbacks receive a list of events.
- Subscriber exceptions are isolated; one failing subscriber must not stop others.
- API event emission happens after the database state is saved.
- API mutations that do not change truth values, such as `set_metadata`, emit no events.
- API mutations that raise before saving emit no events.
- The tests do not assert event ordering for multiple simultaneous changes.

## Error Handling

- `publish` must catch or otherwise isolate subscriber exceptions. In `test_subscriber_exception_logged`, a bad subscriber raises `RuntimeError`, but a later good subscriber still receives the event.
- `test_subscriber_exception_does_not_break_save` confirms that a subscriber exception does not prevent the API mutation from completing or the DB from being updated.
- API exceptions propagate to the caller. `test_no_events_on_exception` expects `api.retract_node("nonexistent", db_path=db)` to raise `KeyError`.
- Because publishing happens after save, an API exception before save prevents both DB mutation and event emission.
- The suite does not test exception behavior inside `compute_changes`; it assumes well-formed dict inputs.

## Topics to Explore

- [file] `reasons/pubsub.py` — Actual implementation of `ChangeEvent`, subscriber registries, dispatch, scoping, and exception logging.
- [function] `reasons/api.py:retract_node` — Shows how cascade retraction computes before/after state and emits dependent-node events.
- [function] `reasons/api.py:add_node` — Demonstrates the simplest API mutation path and subscriber-aware event emission.
- [file] `tests/conftest.py` — Shared pytest configuration and fixtures used across the suite.
- [general] `subscriber-lifecycle` — Explore unsubscribe semantics for scoped subscribers, duplicate subscriptions, thread-safety, and cross-process behavior.

## Beliefs

- `compute-changes-produces-node-changed-events` — `compute_changes` returns `ChangeEvent` objects with `type="node_changed"` for added, deleted, and modified nodes, skipping unchanged entries.
- `changeevent-uses-none-for-absent-state` — New nodes have `old_truth_value=None`; deleted nodes have `new_truth_value=None`.
- `publish-isolates-subscriber-errors` — `publish` catches/logs exceptions from one subscriber and continues dispatching to later subscribers.
- `scoped-subscribers-require-matching-db-path` — A subscriber registered with `db_path` only receives publishes made with that exact `db_path`; global subscribers receive all.
- `api-emits-events-only-after-successful-saves` — API operations publish events after updated truth values are visible through `api.show_node`, and failed mutations emit no events.
