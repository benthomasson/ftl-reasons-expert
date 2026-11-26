---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: `_subscribers` has no lock; explore whether concurrent `subscribe`/`publish` is expected in this codebase.

# `_subscribers` has no lock; is concurrent `subscribe`/`publish` expected?

## Summary

`_subscribers` is a module-global dictionary in `reasons/pubsub.py:35`, and none of the registry operations (`subscribe`, `unsubscribe`, `has_subscribers`, `publish`, `clear`) acquire a lock. The code therefore has no internal thread-safety guarantee.

However, the observed usage strongly suggests that concurrent `subscribe`/`publish` is **not expected** in the current codebase:

- The only production `publish` call is from the synchronous write path in `reasons/api.py:158`.
- `subscribe(...)` is not called anywhere in production code in the observations; all real registration calls appear in `tests/test_pubsub.py`.
- `reasons/pubsub.py` and `reasons/api.py` do not import `threading`, `asyncio`, or other concurrency primitives.
- The pubsub tests are synchronous and use an autouse `clear()` fixture after each test (`tests/test_pubsub.py:18`).
- The module docstring describes this as **in-process** pub/sub after write transactions complete, not as a thread-safe or cross-process event bus.

So the likely contract is: **single-threaded, same-process use**. But that contract is implicit, not enforced.

## What the absence of a lock means

The key implementation details:

- `_subscribers` maps optional `db_path` to a list of callbacks:
  - `reasons/pubsub.py:35`
- `subscribe` mutates the registry:
  - `_subscribers.setdefault(db_path, []).append(callback)` at `reasons/pubsub.py:47`
- `unsubscribe` removes and can pop the key:
  - `reasons/pubsub.py:52-58`
- `publish` snapshots subscriber lists before calling callbacks:
  - `targets = list(_subscribers.get(None, []))` at `reasons/pubsub.py:79`
  - `targets.extend(_subscribers.get(db_path, []))` at `reasons/pubsub.py:80`

Because `publish` copies the lists into `targets`, a callback being added or removed during dispatch will not mutate the list currently being iterated. That reduces one class of failure.

But it does not make registration/dispatch thread-safe:

- `subscribe` uses `setdefault(...).append(...)`, which is a compound operation. A concurrent `unsubscribe` or `clear` can interleave and leave a callback in a list that is no longer referenced by `_subscribers`.
- `publish` performs two separate registry reads: one for global subscribers and one for db-scoped subscribers. A concurrent `unsubscribe` can land between those reads, so the publish snapshot is not a single atomic view.
- A subscriber removed after `publish` copies the list may still receive the current event.
- A subscriber added after `publish` copies the list will not receive the current event.
- `has_subscribers` at `reasons/pubsub.py:61-66` also performs multiple unsynchronized reads, so it is only a best-effort check.
- `clear()` at `reasons/pubsub.py:123` is intended for test teardown. If a publisher is active in another thread, `clear()` can interleave with it.

The CPython GIL makes individual dictionary/list operations atomic enough to avoid interpreter crashes in many cases, but it does **not** make these multi-step operations logically atomic. The code has races, not necessarily memory corruption.

## Where publish is used in production

The observed production integration is:

- `reasons/api.py:139`: `if write and pubsub.has_subscribers(db_path):`
- `reasons/api.py:158`: `pubsub.publish(events, db_path)`

That means the API write path checks whether anyone is listening, computes changes if needed, and publishes after the write transaction. The pubsub docstring in `reasons/pubsub.py` says events are emitted after `_with_network.__exit__` completes, i.e. after save, not during intermediate cascade states.

This is a synchronous write-path integration. The observations do not show any server, MCP handler, or thread pool calling `subscribe` or `publish` concurrently. The concurrency imports that do exist are elsewhere:

- `reasons/llm.py:18`: `import threading`
- `reasons/mcp_client.py:9`: `import asyncio`
- `reasons/mcp_client.py:11`: `import threading`
- `reasons/repair_premises.py:10`: `from concurrent.futures import ThreadPoolExecutor, as_completed`

There is no evidence in the provided observations that those modules call into `reasons.pubsub`. Even if they invoke the API, the API’s write path would need to be called from multiple threads in the same process for the `_subscribers` races to matter.

## Conclusion

The current codebase appears to assume single-threaded access for `reasons.pubsub`. The lack of a lock is consistent with that assumption.

But the module is publicly callable, and nothing in `reasons/pubsub.py` documents “not thread-safe.” If future code introduces concurrent API writes, background subscribers, or a server that calls `subscribe`/`unsubscribe` while writes are happening, this registry should either:

1. add an internal `threading.RLock` around registry mutation and snapshotting; or
2. explicitly document that callers must serialize `subscribe`, `unsubscribe`, `clear`, and `publish`.

The observations are not sufficient to prove the entire runtime is single-threaded because the full `reasons/api.py`, server entrypoints, and import graph are not shown. But the visible usage points strongly toward a synchronous, in-process design.

## Topics to Explore

- [function] `reasons/api.py:_with_network` — Inspect the write transaction wrapper to confirm whether writes can overlap in the same process and exactly where `pubsub.publish` is called after save.
- [file] `reasons/mcp_client.py` — Check whether the async/threaded MCP client ever calls `reasons.api` or `reasons.pubsub` from worker threads.
- [file] `reasons/repair_premises.py` — The `ThreadPoolExecutor` usage may call API write functions concurrently; verify whether those writes can reach `pubsub.publish`.
- [general] `threading-call-graph` — Search for `Thread`, `ThreadPoolExecutor`, `run_in_executor`, `asyncio.to_thread`, and server entrypoints that invoke `api.add_node`, `api.retract_node`, or other write APIs.
- [file] `tests/test_pubsub.py` — Review whether existing tests cover any concurrency, or whether the suite confirms the intended single-threaded usage model.

## Beliefs

- `_subscribers-is-a-module-global-dict` — `_subscribers` is a module-level dictionary keyed by optional `db_path`, shared by all callers in the process (`reasons/pubsub.py:35`).
- `subscribe-unsubscribe-publish-are-unsynchronized` — None of `subscribe`, `unsubscribe`, `has_subscribers`, `publish`, or `clear` acquires a lock in `reasons/pubsub.py`.
- `publish-snapshots-subscriber-lists` — `publish` copies the global and db-scoped subscriber lists into `targets` before invoking callbacks (`reasons/pubsub.py:79-80`).
- `production-code-does-not-call-subscribe` — The observed production code contains no `subscribe(...)` calls; subscriber registration appears only in `tests/test_pubsub.py`.
- `api-write-path-invokes-publish` — The only observed production `pubsub.publish` call is from `reasons/api.py:158` after a write path has checked `has_subscribers` at `reasons/api.py:139`.
