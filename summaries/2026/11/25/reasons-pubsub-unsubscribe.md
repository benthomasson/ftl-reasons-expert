---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Function: unsubscribe in reasons/pubsub.py

## Purpose

`unsubscribe` is the removal counterpart to `subscribe` in the in-process pub/sub registry. It removes a previously registered callback from the module-level `_subscribers` dictionary, scoped by `db_path`. It is used when a consumer no longer wants truth-value change events, typically during teardown or lifecycle cleanup.

The function is intentionally quiet: attempting to remove a callback that is not registered does not raise.

## Contract

- **Preconditions**
  - `callback` should be a value previously passed to `subscribe`, although this is not required or checked.
  - `db_path` should be the same key used when subscribing: `None` for the global subscriber list, or the exact database path string for a database-scoped subscription.
  - `_subscribers` is assumed to be the module-level `dict[db_path, list[Subscriber]]`.

- **Postconditions**
  - If an element equal to `callback` exists in `_subscribers[db_path]`, exactly one matching occurrence is removed via `list.remove`.
  - If that removal makes the list empty, the `db_path` key is removed from `_subscribers`.
  - If `callback` is not present under `db_path`, no callback is removed and no exception is raised.
  - Other `db_path` scopes are untouched.
  - The global list (`db_path=None`) and a database-specific list are independent subscriptions.

- **Important invariant / behavior**
  - `list.remove` removes only the first matching element. If the same callback was registered multiple times under the same `db_path`, a single `unsubscribe` call removes only one registration.
  - Removal uses equality (`==`), not identity (`is`). For ordinary function objects this usually behaves like identity, but for callable objects or bound methods, equality semantics can matter.
  - The function does not verify that `db_path` corresponds to a real database or that paths are normalized. Callers must use the exact same key string.

## Parameters

- `callback: Subscriber`
  - The callable to remove.
  - Expected to be compatible with `Callable[[list[ChangeEvent]], None]`.
  - If not present, the call is a no-op with respect to subscriber removal.

- `db_path: str | None = None`
  - Selects which subscriber list to modify.
  - `None` means the global subscriber list.
  - A string means the database-specific subscriber list for that exact key.
  - If omitted while the callback was registered with a specific `db_path`, the call will not remove that database-scoped subscription.

## Return Value

Always returns `None`.

The caller cannot determine whether a callback was actually removed. A missing callback and a successful removal both return normally. This is intentional for simple cleanup code, but means callers should not rely on `unsubscribe` as a status-checking API.

## Algorithm

1. Look up the subscriber list for `db_path`:
   ```python
   subs = _subscribers.get(db_path, [])
   ```
   If the key exists, `subs` is the actual stored list. If it does not exist, `subs` is a temporary empty list that is not inserted into `_subscribers`.

2. Try to remove one matching callback:
   ```python
   subs.remove(callback)
   ```
   This mutates the stored list in place if the key exists and a match is found.

3. If no matching callback is found, `list.remove` raises `ValueError`, which is swallowed:
   ```python
   except ValueError:
       pass
   ```

4. If the resulting list is empty, remove the key from the registry:
   ```python
   if not subs:
       _subscribers.pop(db_path, None)
   ```
   This cleans up both newly empty lists and pre-existing empty lists for that key. If the key never existed, `pop(..., None)` is a no-op.

## Side Effects

- Mutates the module-global `_subscribers` dictionary and/or one of its lists.
- May remove the `db_path` key when its subscriber list becomes empty.
- Does not invoke the callback.
- Does not perform I/O.
- Does not log.
- Is not synchronized; there is no locking around shared global state.

## Error Handling

- `ValueError` from `list.remove` when the callback is absent is explicitly swallowed.
- `KeyError` is avoided in the cleanup step because `pop(db_path, None)` supplies a default.
- Other exceptions are not handled. In normal use they should not occur, but they could occur if module invariants are violated, e.g. if `_subscribers[db_path]` is not a mutable list, or if a custom callable’s `__eq__` raises during `list.remove`.

## Usage Patterns

Typical usage:

```python
subscribe(my_handler, db_path="/data/tms.db")
...
unsubscribe(my_handler, db_path="/data/tms.db")
```

Usage notes:

- To remove a database-scoped subscription, pass the same `db_path` used during `subscribe`.
- To remove a global subscription, pass `db_path=None` or omit it.
- Calling `unsubscribe` for an absent callback is safe.
- If callbacks may be registered multiple times, call `unsubscribe` once per registration or use `clear()` to remove all subscribers.
- Because `publish` snapshots subscriber lists before dispatch, unsubscribing during a publish callback does not affect the already-snapshotted callbacks for that ongoing publish.

## Dependencies

- `Subscriber` from the same module, which is an alias for `Callable[[list[ChangeEvent]], None]`.
- The module-level `_subscribers` registry.
- Standard Python list and dict operations only.
- No external libraries or I/O dependencies.

## Assumptions Not Enforced by the Type System

- `db_path` values are not normalized; callers must use exactly the same `None` or string key.
- `callback` removal uses equality, not identity, so callable objects with custom `__eq__` can behave unexpectedly.
- Duplicate subscriptions are allowed by `subscribe`; `unsubscribe` removes only one occurrence.
- `_subscribers` is assumed to contain lists, not arbitrary iterables.
- The function assumes no concurrent mutation of `_subscribers` by another thread.
- The function does not validate that `callback` is actually a valid `Subscriber`.

## Topics to Explore

- [file] `tests/test_pubsub.py` — Covers subscribe/unsubscribe behavior, likely including duplicate registration, db scoping, and error handling.
- [function] `reasons/pubsub.py:subscribe` — Registration counterpart; explains `setdefault(...).append(...)` and duplicate-registration semantics.
- [function] `reasons/pubsub.py:publish` — Shows how global and database-scoped subscribers are selected and invoked.
- [function] `reasons/pubsub.py:has_subscribers` — Demonstrates how global and db-specific subscription lists are queried.
- [general] `pubsub-db-path-scoping` — Explores the exact-match `db_path` keying model and how `None` differs from database-specific subscriptions.

## Beliefs

- `unsubscribe-removes-first-matching-callback` — A single call removes only the first element equal to `callback` under the selected `db_path`.
- `missing-subscriber-is-silent` — Unsubscribing an absent callback swallows `ValueError` and returns normally.
- `empty-db-path-key-is-cleaned` — When the target subscriber list becomes empty, the corresponding `db_path` key is removed from `_subscribers`.
- `db-path-must-match-subscription-scope` — A callback subscribed under a specific `db_path` is not removed by unsubscribing with a different `db_path`.
- `unsubscribe-is-not-thread-safe` — The function mutates shared global dictionary/list state without synchronization.
