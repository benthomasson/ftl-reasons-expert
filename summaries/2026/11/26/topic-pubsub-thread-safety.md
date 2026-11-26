---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Whether concurrent subscribe/unsubscribe/publish is expected and how the lack of locking affects callers.

## Short answer

Concurrent `subscribe` / `unsubscribe` / `publish` is **not declared or supported** by the code as written. `reasons/pubsub.py` is a plain in-process registry with no `Lock`, `RLock`, queue, or other synchronization. The safest interpretation is: **treat this module as not thread-safe unless callers provide external synchronization**.

The code may still “work” under the CPython GIL for simple operations, but the compound operations are not atomic, and there are real logical races. Also, the observations do not include the full `_with_network`/DB writer implementation, so we cannot prove whether API-level writes can actually call `publish` concurrently; what we can say is that `pubsub` itself does nothing to make that safe.

## Evidence

- `_subscribers` is a plain global mutable dict: `reasons/pubsub.py:35`.
- `subscribe` mutates it directly with `setdefault(...).append(...)`: `reasons/pubsub.py:47`.
- `unsubscribe` does `get` / `remove` / conditional `pop` without a lock: `reasons/pubsub.py:52-58`.
- `has_subscribers` reads the same dict without a lock: `reasons/pubsub.py:63-66`.
- `publish` copies the relevant lists and then invokes callbacks synchronously: `reasons/pubsub.py:79-84`.
- The write path checks `pubsub.has_subscribers(db_path)` at `reasons/api.py:139` and later calls `pubsub.publish(events, db_path)` at `reasons/api.py:158`.
- The concurrency search shows locks elsewhere, e.g. `reasons/llm.py:57`, but no locking in `reasons/pubsub.py`.
- The docs search for `concurrent|thread.?safe|thread safety|locking|subscribe|unsubscribe|publish` returned no matches, so there is no documented concurrency contract.
- The visible `tests/test_pubsub.py` tests are sequential and do not exercise concurrent registration/publishing.

## How the lack of locking affects callers

### 1. Registration can be lost under concurrent subscribe/unsubscribe

`subscribe` and `unsubscribe` perform multi-step operations against the same dict/list. For example:

1. Thread A calls `subscribe` and `setdefault(db_path, [])` returns an existing list.
2. Thread B calls `unsubscribe`, removes the last callback, sees the list is empty, and does `_subscribers.pop(db_path, None)`.
3. Thread A appends its callback to the now-orphaned list.

Result: the subscriber appears to have subscribed successfully, but it is no longer in `_subscribers`. That is a real race caused by `reasons/pubsub.py:47` and `reasons/pubsub.py:52-58` not being coordinated.

### 2. `unsubscribe` does not cancel an in-flight `publish`

`publish` snapshots the global and DB-specific subscriber lists at `reasons/pubsub.py:79-80`, then iterates the snapshot at `reasons/pubsub.py:82-84`. If another thread unsubscribes after the snapshot is taken, the callback may still run for that batch. So callers should not assume that once `unsubscribe(cb)` returns, `cb` will never be invoked again if a publish is concurrently in progress.

### 3. A subscriber may miss a transaction if it races `has_subscribers`

The API layer only computes and publishes changes if `has_subscribers(db_path)` is true at `reasons/api.py:139`. That check is not atomic with the later `publish` at `reasons/api.py:158`. If a thread subscribes just after the check returns false, it can miss the current transaction’s events. If a thread unsubscribes after the check returns true, the later publish may have no target. This is timing-dependent and not a crash, but it violates any assumption of “subscribe before write means receive that write’s event.”

### 4. Concurrent `publish` calls can invoke subscriber callbacks concurrently

`publish` does not serialize calls. If two threads call `publish` at the same time, the same callback can be invoked concurrently with different event lists. A subscriber that mutates shared state must therefore be thread-safe, or all publish calls must be externally serialized. In addition, callbacks run synchronously on the publishing thread, so a slow subscriber blocks the publisher; there is no background delivery queue.

### 5. Subscriber exceptions are swallowed and logged

`publish` catches exceptions per subscriber and logs them at `reasons/pubsub.py:85-86`. That prevents one bad subscriber from breaking the publish loop, but it also means a caller of `publish` cannot observe subscriber failure. In concurrent settings this remains true: failures only appear in logs.

## What is missing from the observations

The full `_with_network` implementation and the PostgreSQL writer implementation are not shown. Those are needed to determine whether the application actually produces concurrent `publish` calls from concurrent writes. The `reasons/pg.py` docstring says it “Enables concurrent writers,” but the provided observations do not show whether those writers reach `pubsub.publish` concurrently or whether some higher layer serializes them.

## Recommendation

If concurrent subscribe/unsubscribe/publish is required, do not rely on the current `reasons/pubsub.py` behavior. Either:

- document and enforce that all pubsub mutations/publishes must happen under an external lock, or
- change the pubsub module to use a lock around registry mutation and snapshotting, while avoiding holding the lock while invoking subscriber callbacks, or
- move to a copy-on-write immutable registry or an explicit event queue.

At minimum, callers should treat registration and delivery as best-effort under concurrency and should not assume “unsubscribe means no further callbacks” or “subscribe means receive the next transaction” without external synchronization.

## Topics to Explore

- [file] `reasons/api.py` — Inspect `_with_network` and the write/commit path around `has_subscribers` and `publish` to see whether concurrent writers can publish concurrently.
- [function] `reasons/api.py:_with_network` — Shows the transaction boundary where events are batched and published.
- [file] `reasons/pg.py` — The docstring claims concurrent writers; verify whether that concurrency reaches `pubsub.publish`.
- [function] `reasons/pubsub.py:publish` — Review the snapshot-and-callback behavior and whether a lock/queue redesign is needed.
- [general] `pubsub-concurrency-contract` — Decide whether to document non-thread-safety, add external locking, or redesign the registry.

## Beliefs

- `pubsub-has-no-locking` — `reasons/pubsub.py` has no `Lock`/`RLock`/`threading` usage; `_subscribers` is a plain global dict mutated by `subscribe` and `unsubscribe`.
- `publish-runs-callbacks-synchronously` — `publish` copies target lists and invokes each subscriber inline on the publishing thread; subscriber exceptions are caught and logged.
- `unsubscribe-does-not-prevent-in-flight-delivery` — Because `publish` snapshots lists before invoking callbacks, a callback copied into `targets` may run after a concurrent `unsubscribe` returns.
- `api-has-subscribers-check-is-not-atomic-with-publish` — `reasons/api.py` checks `pubsub.has_subscribers(db_path)` at line 139 and calls `publish` later at line 158, so concurrent registration changes can cause missed or skipped event delivery.
- `pubsub-concurrency-contract-is-undocumented` — The documentation search found no pubsub concurrency or thread-safety references, and the visible tests are sequential.
