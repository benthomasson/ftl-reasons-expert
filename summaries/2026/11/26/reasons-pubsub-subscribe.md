---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Function: subscribe in reasons/pubsub.py

## `reasons/pubsub.py:subscribe`

### Purpose
`subscribe` registers a callback with the module’s in-process publish/subscribe registry. It exists so callers can be notified after write transactions produce truth-value change events. Registration is purely in-memory and process-local; it does not persist across processes or restarts.

A callback registered with `db_path=None` becomes a global subscriber and receives events for all databases. A callback registered with a concrete `db_path` receives only events whose `publish(..., db_path=...)` value exactly equals that key.

### Contract
- **Precondition:** `callback` should be a callable accepting `list[ChangeEvent]`. `db_path` should be a string or `None`.
- **Postcondition:** On successful return, `callback` has been appended to `_subscribers[db_path]`. If no list existed for that key, an empty list is created first via `setdefault`.
- **Return:** Always returns `None`.
- **Invariants assumed by the module:** `_subscribers` maps `str | None` keys to lists of subscribers. Registration order is preserved within each list. `publish` later reads the `None` key for global subscribers and the concrete `db_path` key for database-specific subscribers.
- **Not enforced by the type system:**
  - `callback` is not validated at runtime; non-callables are accepted and only fail later during `publish`.
  - Duplicate registrations are allowed. The same callback may be appended multiple times.
  - The same callback can be both global and database-specific, causing it to be invoked twice for matching database events.
  - `db_path` is not normalized. `"foo.db"` and `"./foo.db"` are different subscription keys.
  - Subscribers are stored by strong reference; registered callbacks and their captured objects can remain alive until `unsubscribe` or `clear`.
  - The `events` list passed at publish time is shared across subscribers; `subscribe` does not enforce that callbacks treat it as read-only.

### Parameters
- `callback: Subscriber`
  - Type alias: `Callable[[list[ChangeEvent]], None]`.
  - Registered but not invoked by `subscribe`.
  - May raise when later invoked; `publish` catches and logs subscriber exceptions.
  - A non-callable value is accepted by `subscribe`, but will fail when `publish` attempts to call it.

- `db_path: str | None = None`
  - `None` registers a global subscriber that receives events from all databases.
  - A string registers a database-specific subscriber for events whose `db_path` exactly matches this string.
  - An empty string is a valid distinct key, not equivalent to `None`.
  - No path canonicalization or validation is performed.
  - If a non-string, non-`None` hashable value is passed at runtime, it may be accepted as a dict key but will not match normal `publish` lookups. Unhashable values raise `TypeError`.

### Return Value
`subscribe` returns `None` implicitly. There is no success flag, registration ID, or error return. Callers must retain the original callback and original `db_path` value if they intend to call `unsubscribe` later.

### Algorithm
1. Look up `db_path` in the module-level `_subscribers` dictionary using `setdefault`.
2. If the key is absent, insert a new empty list and return that list.
3. If the key is present, return the existing list.
4. Append `callback` to that list.
5. Return `None`.

Equivalent logic:

```python
_subscribers.setdefault(db_path, []).append(callback)
```

No copying, filtering, deduplication, validation, or immediate notification occurs.

### Side Effects
- Mutates the global `_subscribers` dictionary.
- Appends to a mutable list stored in that dictionary.
- Holds a strong reference to `callback`.
- Affects future `publish` calls: the callback becomes eligible for dispatch after matching events.
- Does not invoke the callback immediately.
- Does not perform I/O, logging, persistence, or network calls.
- Interacts with `unsubscribe`, `has_subscribers`, `publish`, and `clear`, all of which read or mutate the same module-level registry.

### Error Handling
`subscribe` has no explicit error handling. Normally it does not raise for valid inputs. Possible failures include:
- `TypeError` if `db_path` is unhashable at runtime.
- `AttributeError` if `_subscribers[db_path]` is somehow not a list, which would break the module invariant.
- `MemoryError` under resource exhaustion.

It does **not** validate that `callback` is callable. A bad callback is accepted and only fails later in `publish`, where the exception is caught and logged via `logger.exception`. `subscribe` itself does not catch or log anything.

### Usage Patterns
Typical usage:

```python
def on_events(events: list[ChangeEvent]) -> None:
    for event in events:
        ...

subscribe(on_events)                 # global subscriber
subscribe(on_events, db_path="/db/a")  # database-specific subscriber
```

Callers should:
- Register before the write transaction whose events they want to observe.
- Use the exact same `db_path` string when unsubscribing.
- Avoid duplicate registrations unless duplicate delivery is intended.
- Unsubscribe or call `clear()` during teardown to avoid stale references.
- Treat the `events` list as read-only inside callbacks, because `publish` passes the same list to multiple subscribers.
- Be aware that `subscribe` is not thread-safe; concurrent mutation of `_subscribers` is not synchronized.

### Dependencies
`subscribe` depends on:
- The module-level `_subscribers` dictionary.
- The `Subscriber` type alias, defined as `Callable[[list[ChangeEvent]], None]`.
- `ChangeEvent` for the callback type signature.
- Python’s built-in `dict.setdefault` and `list.append`.

It does not directly use the imported `logging` module, although `publish` does.

## Topics to Explore
- [file] `reasons/pubsub.py` — Contains the full subscription lifecycle: registration, unsubscription, dispatch, change computation, and test cleanup.
- [function] `reasons/pubsub.py:publish` — Shows how global and database-specific subscribers are combined, invoked, and isolated from callback exceptions.
- [function] `reasons/pubsub.py:unsubscribe` — Explains removal semantics, exact-key matching, and empty-list cleanup.
- [file] `tests/test_pubsub.py` — Covers expected behavior for global subscribers, database filtering, duplicate registrations, and event dispatch.
- [general] `in-process-pubsub-lifecycle` — Worth exploring for thread-safety, strong-reference retention, duplicate delivery, and callback isolation concerns.

## Beliefs
- `subscribe-is-process-local` — `subscribe` only mutates the in-memory `_subscribers` dictionary and does not persist subscriptions across processes or restarts.
- `global-subscribers-receive-all-db-events` — A callback registered with `db_path=None` is stored under the `None` key and is included in every `publish` dispatch regardless of database.
- `specific-subscribers-require-exact-db-path-match` — A callback registered with a string `db_path` receives only events whose `publish` `db_path` is exactly equal to that string.
- `subscribe-allows-duplicate-registrations` — Registering the same callback more than once for the same `db_path` appends duplicate entries and causes multiple invocations.
- `subscribe-does-not-validate-callables` — `subscribe` accepts any object as `callback`; non-callable callbacks fail later in `publish`, where exceptions are logged and swallowed.
