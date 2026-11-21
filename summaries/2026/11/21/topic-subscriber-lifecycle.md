---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Explore unsubscribe semantics for scoped subscribers, duplicate subscriptions, thread-safety, and cross-process behavior.

## Summary

`reasons/pubsub.py` implements a **process-local, in-memory subscriber registry**. `subscribe()` appends callbacks to a list keyed by `db_path` (`None` means global) at `reasons/pubsub.py:38`, and `unsubscribe()` removes from exactly one key at `reasons/pubsub.py:50`. There is **no deduplication**, **no locking**, and **no cross-process delivery mechanism**.

## Scoped unsubscribe semantics

- The registry is `_subscribers: dict[str | None, list[Subscriber]]`; the key is the exact `db_path` string, or `None` for global subscribers.
- `unsubscribe(callback, db_path="x.db")` removes the callback only from the `"x.db"` bucket.
- `unsubscribe(callback)` uses the default `db_path=None`, so it removes only from the **global** bucket. It does **not** remove the callback from scoped buckets.
- If the same callback is registered both globally and for `"x.db"`, you need two separate unsubscribe calls to fully remove it:
  - `unsubscribe(cb, db_path="x.db")` leaves the global registration intact.
  - `unsubscribe(cb)` leaves the scoped registration intact.
- `unsubscribe()` uses `subs.remove(callback)` at `reasons/pubsub.py:54`, so it removes only the **first matching occurrence**. If the same callback was subscribed twice under the same scope, one unsubscribe leaves one registration.
- Missing callbacks are silently ignored: `ValueError` is swallowed at `reasons/pubsub.py:55-56`. The function returns `None`, so callers cannot tell whether removal actually happened.
- When the last subscriber for a scope is removed, the bucket is deleted with `_subscribers.pop(db_path, None)` at `reasons/pubsub.py:58`.
- `clear()` removes all subscribers from all scopes.

Related caveat: `has_subscribers()` with no `db_path` checks only the global `None` bucket. A scoped-only subscriber does **not** make `has_subscribers()` return `True`.

## Duplicate subscriptions

`subscribe()` appends unconditionally:

- `subscribe()` uses `_subscribers.setdefault(db_path, []).append(callback)` at `reasons/pubsub.py:38` / line 47.
- There is no membership check, so duplicate subscriptions are allowed.
- `publish()` builds targets by copying global subscribers and extending with the exact scoped list at `reasons/pubsub.py:79-80`, with no deduplication.
- Therefore:
  - Same callback subscribed twice under the same `db_path` → called twice.
  - Same callback subscribed globally and under `"x.db"` → called twice for events published with `"x.db"`.
  - Same callback subscribed under two different scoped buckets → only the matching scoped bucket plus global are used for a given `publish(..., db_path=...)`.
- `publish()` dispatches global subscribers first, then the exact `db_path` subscribers.
- Because `publish()` iterates over a copied `targets` list, registry changes made during dispatch affect future publishes, not the current copied batch.
- Subscriber exceptions are caught per invocation and logged at `reasons/pubsub.py:82-86`; one failing subscriber does not prevent later subscribers from running.

## Thread-safety

The pubsub registry is **not thread-safe**:

- `reasons/pubsub.py` imports only `logging`, `dataclass`, and `Callable`; there are no locks, queues, or synchronization primitives.
- The thread/process search in the observations found no `threading`, `Lock`, `multiprocessing`, `Queue`, `redis`, `LISTEN`, or `NOTIFY` usage in `reasons/pubsub.py`.
- `subscribe()`, `unsubscribe()`, `clear()`, `publish()`, and `has_subscribers()` mutate or read shared module-level state without coordination.
- Race examples include:
  - `unsubscribe()` removing the last item and popping the bucket while another thread calls `subscribe()` on the same bucket.
  - `publish()` copying a list while another thread mutates that list.
  - `clear()` wiping the registry while `publish()` is running.
- The GIL does not make these compound operations safe at the application level.
- Dispatch is synchronous: subscribers run on the thread that calls `publish()`, after save completes. A slow subscriber blocks that writer thread.

## Cross-process behavior

There is **no cross-process pubsub behavior**:

- Subscribers live in the module-level `_subscribers` dict inside one Python interpreter/process.
- `db_path` is only a dictionary key; it is not a shared broker, file watcher, database notification channel, or IPC mechanism.
- If process A subscribes and process B writes to the same database path, process B’s `publish()` only sees process B’s local registry. Process A will not receive the event.
- The compatibility module `reasons_lib/pubsub.py:1` simply re-exports `reasons.pubsub` functions via `from reasons.pubsub import *`, so both import paths share the same in-process registry, not a cross-process one.

## Test coverage and gaps

Existing tests cover basic global unsubscribe in `tests/test_pubsub.py:test_unsubscribe` around lines 83–88, scoped publish filtering, and global-plus-scoped delivery with two different callbacks.

The observations do **not** show tests for:

- Unsubscribing a scoped callback while leaving a global registration.
- Duplicate subscriptions of the same callback.
- Thread-safety races.
- Cross-process delivery.
- The exact `_with_network.__exit__` call site and how `has_subscribers(db_path)` is used in production.

## Topics to Explore

- [file] `reasons/pubsub.py` — Full registry, scoped unsubscribe, duplicate handling, and publish dispatch logic.
- [function] `reasons/pubsub.py:unsubscribe` — Exact scoped-removal, first-occurrence removal, silent miss handling, and empty-bucket cleanup.
- [function] `reasons/pubsub.py:publish` — Global-plus-scoped target assembly, duplicate invocation behavior, and exception isolation.
- [general] `with-network-pubsub-call-site` — Inspect `_with_network.__exit__` to see when `publish()` and `has_subscribers()` are called and with which `db_path`.
- [general] `cross-process-pubsub` — Current design needs an external broker, DB notification mechanism, or polling layer to support cross-process subscribers.

## Beliefs

- `pubsub-registry-is-process-local` — The subscriber registry is the module-level `_subscribers` dict in `reasons/pubsub.py`, so subscriptions are not shared across processes.
- `unsubscribe-targets-exact-scope` — `unsubscribe(callback, db_path=...)` removes only from that exact `db_path` bucket; omitting `db_path` targets only the global `None` bucket.
- `duplicate-subscriptions-are-not-deduplicated` — `subscribe()` appends every registration and `publish()` concatenates global and scoped lists without deduplication, so duplicate registrations cause duplicate callbacks.
- `pubsub-lacks-thread-synchronization` — `reasons/pubsub.py` has no locks or synchronization, so concurrent registry mutation and dispatch are not thread-safe.
- `subscriber-exceptions-do-not-propagate` — `publish()` catches and logs exceptions from each subscriber, so one subscriber failure does not break the write path.
