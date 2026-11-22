---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: publish in reasons/pubsub.py

# `publish` — Explanation

## 1. Purpose

`publish` is the dispatch half of an in-process, process-wide publish/subscribe mechanism for TMS truth-value change events. It takes a batch of `ChangeEvent`s (produced by `compute_changes` after a write transaction) and invokes every registered subscriber that is interested in the given `db_path`, plus every "global" subscriber registered with `None`.

It exists so that after a `_with_network` write transaction commits (the docstring says it is called from `_with_network.__exit__` after `save`), the module can notify listeners without coupling callers to any particular consumer. Errors in consumers are isolated so a bad subscriber cannot abort a database commit.

## 2. Contract

- **Precondition:** `events` is a `list[ChangeEvent]`. If empty, the call is a no-op.
- **Precondition:** `db_path` identifies the database the events belong to. Subscribers registered for this exact key, plus those registered under `None`, will be invoked.
- **Postcondition:** Every subscriber that was registered for `None` or for `db_path`, and that was snapshotted at call time, has been invoked with `events` (unless an earlier subscriber raised, in which case only that subscriber is skipped).
- **Postcondition:** No exception originating inside a subscriber is propagated to the caller.
- **Invariants:**
  - Iteration is over a **snapshot** (`list(...)` + `extend(...)`); mutations to `_subscribers` during dispatch do not affect the current call.
  - All subscribers in the snapshot receive the *same* `events` list object.
  - Subscribers may be invoked in this order: global subscribers first (insertion order), then `db_path`-specific subscribers (insertion order). No cross-list ordering guarantee.

## 3. Parameters

- **`events: list[ChangeEvent]`**
  - The batched changes. Only falsy (empty) is special-cased; non-list iterables are not handled (the type is enforced only by annotation). If a subscriber mutates the list, later subscribers in the same dispatch will observe the mutation.
  - **Edge case:** events whose own `ChangeEvent.db_path` does not match the `db_path` argument are **not filtered out or re-routed** — `publish` trusts the caller to have scoped events correctly.
- **`db_path: str`**
  - Key into `_subscribers` for per-database subscriptions.
  - **Edge case:** the type is `str`, but `subscribe` accepts `None` for `db_path`. If a caller passes `None` to `publish`, then `targets` becomes `list(_subscribers[None])` extended with `_subscribers[None]` again — global subscribers are invoked **twice**. This is not prevented by the type system or at runtime.

## 4. Return Value

- Returns `None` implicitly. Callers receive no signal about which subscribers succeeded or failed; success/failure is only observable in the log.

## 5. Algorithm

1. **Early exit:** if `events` is empty (`if not events: return`), return before touching `_subscribers`. This avoids snapshotting subscribers when there is no work.
2. **Snapshot globals:** `targets = list(_subscribers.get(None, []))` — copy the global-subscriber list. `list()` is important so that if a subscriber calls `subscribe`/`unsubscribe` for the `None` key mid-dispatch, the ongoing iteration is unaffected.
3. **Append path-specific:** `targets.extend(_subscribers.get(db_path, []))` — adds every subscriber registered for this exact `db_path` to the snapshot. `.get(db_path, [])` gracefully handles databases with no subscribers.
4. **Dispatch:** iterate `targets`, calling `subscriber(events)` per element, wrapped in `try/except Exception`. On exception, `logger.exception(...)` records it with traceback and the loop proceeds to the next subscriber.

Note the snapshot is only *shallow* — the list of subscribers is stable, but the `events` list itself is passed by reference and is shared.

## 6. Side Effects

- Invokes arbitrary user-supplied callbacks (`Subscriber = Callable[[list[ChangeEvent]], None]`). Their side effects (DB writes, logging, mutation of `events`, further `publish` calls, etc.) are outside `publish`'s control.
- Emits log records via `logger.exception` when a subscriber raises.
- Reads module-level state `_subscribers` — the function does **not** mutate it.
- No I/O, no file access, no network calls, no mutation of `events` by `publish` itself.

## 7. Error Handling

- **Empty `events`:** silently returns (not an error).
- **Subscriber raises `Exception`** (or any subclass except `BaseException`-only ones): caught, logged with `logger.exception`, dispatch continues. This is the primary robustness guarantee — one broken subscriber cannot break the transaction or starve subsequent subscribers.
- **Subscriber raises `BaseException` subclass** (`KeyboardInterrupt`, `SystemExit`, `GeneratorExit`, or a custom one): **propagates**. The bare `except Exception` does not catch these.
- **Reentrancy:** a subscriber that calls `publish` recursively is not guarded against — infinite recursion or reordering is possible and is not detected.
- **`db_path=None` misuse:** double invocation of globals (see §3), silently.

## 8. Usage Patterns

- Called once per completed write transaction from `_with_network.__exit__` (per the docstring), after a successful `save`.
- The pipeline is: `_with_network` captures before/after truth-value snapshots → `compute_changes` builds `ChangeEvent`s → `publish(events, db_path)` fans them out.
- Callers can cheaply skip the work when nobody is listening by checking `has_subscribers(db_path)` before computing and publishing.
- Subscriber obligations:
  - Should be reasonably fast (they run synchronously on the writer's call stack).
  - Should tolerate being invoked with the same `events` list object shared with other subscribers.
  - Should not depend on being invoked exactly once — a subscriber registered under both `None` and `db_path` will be called twice for the same events.
  - Should be defensive about mutations of `events` by earlier subscribers.
- Lifecycle is process-global: subscriptions persist until `unsubscribe` or `clear()` (intended for test teardown).

## 9. Dependencies

- `logging` — uses the module-level `logger = logging.getLogger(__name__)`; `logger.exception` must be called inside an `except` block.
- `ChangeEvent` (dataclass in the same module) — only structurally used; `publish` does not inspect the event fields.
- `Subscriber` type alias and `_subscribers` module-level dict — the actual fan-out source.
- Caller (`_with_network`) is an external dependency at the behavioral level, not imported here.

## Assumptions not enforced by the type system

- `db_path` is never `None` when `publish` is called, even though `subscribe` accepts `None`.
- `events` all correspond to the supplied `db_path`.
- Subscribers are safe to run on the writer's thread (no concurrency guards exist).
- Subscribers do not raise `BaseException` subclasses, do not recurse into `publish`, and do not mutate `events` in ways that harm later subscribers.
- Subscribers do not need deduplication between the `None` and per-`db_path` registries.

## Topics to Explore

- [function] `reasons/pubsub.py:compute_changes` — Produces the `ChangeEvent` batch that feeds `publish`; explains the "before/after snapshot" contract referenced in the module docstring.
- [function] `reasons/pubsub.py:subscribe` — The registration contract, especially the `None` vs. `db_path` keying that `publish` depends on.
- [general] `_with_network lifecycle` — The caller that triggers `publish` on `__exit__` after save; where before/after snapshots are captured.
- [file] `tests/test_pubsub.py` — Likely covers empty-events fast path, exception isolation, and global vs. per-db routing; use it to verify behavior expectations.
- [general] `thread-safety-of-pubsub` — `_subscribers` has no lock; explore whether concurrent `subscribe`/`publish` is expected in this codebase.

## Beliefs

- `publish-returns-immediately-on-empty-events` — An empty `events` list causes `publish` to return before reading `_subscribers`.
- `subscriber-exceptions-are-swallowed-not-propagated` — Exceptions raised by subscribers are caught, logged via `logger.exception`, and never propagate to the caller.
- `global-subscribers-received-by-every-publish-call` — Subscribers registered with `db_path=None` are invoked on every `publish` call, regardless of the `db_path` argument.
- `dispatch-iterates-a-snapshot-of-subscribers` — `publish` iterates a copied list (`list(...)` + `extend(...)`), so `_subscribers` mutations made by subscribers do not affect the current dispatch.
- `publish-does-not-filter-events-by-db-path` — `publish` does not inspect `ChangeEvent.db_path`; it relies on the caller to supply events already scoped to `db_path`.
- `db-path-none-causes-double-invocation` — If `db_path=None` reaches `publish`, global subscribers are called twice because `None` is used for both the global list and the per-db lookup.
