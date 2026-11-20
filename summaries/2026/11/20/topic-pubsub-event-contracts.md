---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Understand the events published/subscribed through this module (channels, topics, payload schemas) since the shim's behaviour is entirely determined by that contract.

# Pub/Sub Contract in `reasons.pubsub` (and its shim)

## TL;DR — the "channels" are not topics

This is an **in-process, synchronous pub/sub**, not a broker. There are no topic strings, no channels, and no message queue. The `topics` module (`api.topics`, `tests/test_topics.py`) is completely unrelated — it is word-frequency extraction over node IDs, not a pub/sub channel. Similarly, `mind_service.py` is an HTTP client for a remote belief sync, unrelated to this pub/sub.

The only "channel" dimension is a **`db_path` string** (plus `None`, meaning "all databases / global"). Subscribers are stored per-`db_path` in a module-level dict at `reasons/pubsub.py`:

```python
_subscribers: dict[str | None, list[Subscriber]] = {}
```

The `reasons_lib/pubsub.py` shim is a bare `from reasons.pubsub import *` — it adds no behaviour, so its entire contract is the one described below. Note it has **no `__all__`** in the source module, so the star import exports everything that doesn't start with `_` (e.g. `logger`, `logging`, `dataclass`, `Callable`, `Subscriber` in addition to the intended names).

## Payload schema — one batch per write transaction

```python
@dataclass(frozen=True)
class ChangeEvent:
    type: str
    node_id: str
    old_truth_value: str | None
    new_truth_value: str | None
    db_path: str
```

Facts from code + tests:

| Field | Contract |
|---|---|
| `type` | Always `"node_changed"` — the only value produced by `compute_changes` and the only value asserted in `tests/test_pubsub.py::test_event_shape` / `test_propagate_emits_events` |
| `node_id` | Node identifier string |
| `old_truth_value` | `"IN"`, `"OUT"`, or `None` when the node is newly created |
| `new_truth_value` | `"IN"`, `"OUT"`, or `None` when the node is deleted |
| `db_path` | The DB that produced the change — lets global subscribers disambiguate |

The dataclass is `frozen=True`, so subscribers cannot mutate the payload the shim dispatched.

The payload delivered to a callback is a **`list[ChangeEvent]`**, not a single event — one call per write transaction, containing all truth-value changes from that transaction (`Subscriber = Callable[[list[ChangeEvent]], None]`).

Crucially, the event carries **what** changed but not **why**. The module docstring states this explicitly, and `compute_changes` confirms it: it only diffs a `before` and `after` truth-value snapshot, so `cause` / `cascade_root` metadata never appears. Any consumer that needs those must derive them elsewhere.

## Channel / subscription API

```python
subscribe(callback, db_path=None)     # None == global
unsubscribe(callback, db_path=None)
has_subscribers(db_path=None)         # True if global OR db-scoped subs exist
clear()                               # test teardown
publish(events, db_path)              # dispatch
compute_changes(before, after, db_path) -> list[ChangeEvent]
```

Delivery semantics you must respect:

1. **Fan-out is global ∪ db-scoped.** `publish` copies the global list, then appends the db-scoped list (`targets = list(_subscribers.get(None, []))` then `.extend(...)`). Both a global subscriber and a scoped subscriber for the same `db_path` get the batch — asserted in `tests/test_pubsub.py::test_global_and_scoped_both_receive` (both lists get length 1).
2. **Scoped isolation.** A subscriber registered for `"my.db"` never sees events published with `"other.db"` — `test_db_scoped_subscriber`.
3. **Empty batches are swallowed.** `publish` returns early on `if not events:` — subscribers are never invoked with `[]` (`test_publish_empty_list_skipped`).
4. **Subscriber exceptions are isolated.** Each subscriber runs inside its own `try/except`, logging `"pubsub subscriber raised an exception"` *without* re-raising — so one bad subscriber cannot break the save path (`test_subscriber_exception_logged`, `test_subscriber_exception_does_not_break_save`).
5. **Snapshot-on-dispatch.** Because `targets` is materialised before iteration, unsubscribing inside a callback will not prevent the current dispatch from calling that (now-removed) subscriber.
6. **Duplicate registration is allowed.** `subscribe` appends without deduping; the same `callback` registered twice will be called twice. `unsubscribe` removes only the first match (`list.remove`), and drops the `db_path` bucket when empty.

## Change detection rules (`compute_changes`)

`compute_changes(before, after, db_path)` walks both dictionaries:

- For every `nid` in `after`, if `before.get(nid) != new_tv`, emit an event with that old/new pair. A new node yields `old_truth_value=None`.
- For every `nid` in `before` missing from `after`, emit an event with `new_truth_value=None` (deletion).
- `before[nid] == after[nid]` produces nothing.

Consequences visible in tests:
- `test_no_changes` → empty list
- `test_truth_value_change` → exactly one event with the field values
- `test_new_node`, `test_deleted_node`, `test_new_node_out` → direction encoded via `None`
- `test_multiple_changes` → one event per changed node
- `set_metadata` does **not** change truth value → `test_no_event_when_no_change` receives nothing

So the event stream is a **truth-value delta stream**, not a general mutation stream. Nodes changing only metadata or text are invisible.

## Emission timing (from `reasons/api.py`)

The publishing integration lives in `reasons/api.py` around lines 139 and 158:

```python
# api.py:139
if write and pubsub.has_subscribers(db_path):
    ...
# api.py:158
    pubsub.publish(events, db_path)
```

Behavioural consequences agreed across the doc-string (module header of `reasons/pubsub.py`) and `tests/test_pubsub.py::TestApiIntegration`:

- **Batched per transaction.** One `publish` call per `_with_network` exit, after `save`.
- **Before/after snapshot diff.** Events describe intermediate cascade states as a *single* final delta — no event is emitted for a transient state that is later reverted inside the same transaction. Tests `test_retract_cascade_emits_events` and `test_assert_emits_events` verify that the *final* stable values are all `"OUT"` / `"IN"` for the whole cascade (`a`, `b`, `c`) in one batch.
- **Gated on subscribers.** `has_subscribers(db_path)` is checked before building the diff, so snapshots are not computed when nobody is listening (`test_no_snapshot_without_subscribers`). Global subscribers count — `has_subscribers` returns True if the `None` bucket is non-empty regardless of `db_path`.
- **Post-save visibility.** Subscribers run *after* the transaction commits; `test_events_after_save` re-reads the DB inside the callback and sees the persisted `"IN"`.
- **Rollback suppresses events.** If the write raises, nothing is published — `test_no_events_on_exception` asserts `received == []` when `retract_node("nonexistent")` raises `KeyError`.
- **Only truth-value-touching calls emit.** `add_node`, `retract_node`, `assert_node`, `add_justification`, `challenge`, `defend`, `add_nogood`, and `propagate` can each produce events; `set_metadata` does not.
- **`propagate` on a consistent network is a no-op.** `test_propagate_emits_events` accepts an empty batch — it only asserts that any events present are well-formed.

## Shim contract

`reasons_lib/pubsub.py` (3 lines) is:

```python
"""Backwards-compatibility shim — use ``from reasons.pubsub`` instead."""
from reasons.pubsub import *  # noqa: F401,F403
```

Because there is no `__all__` in `reasons/pubsub.py`, the shim re-exports every public top-level name — `ChangeEvent`, `Subscriber`, `subscribe`, `unsubscribe`, `has_subscribers`, `publish`, `compute_changes`, `clear`, plus incidental imports `logging`, `dataclass`, `Callable`, and `logger`. Nothing in the shim adds behaviour, so its contract is exactly the `reasons.pubsub` contract above.

## Gaps / things not shown

- The body of `_with_network` in `reasons/api.py` (only refs at 139 and 158) — I cannot see how `before` / `after` snapshots are built, nor whether events are suppressed on certain exception paths beyond the one tested.
- Whether the Postgres dispatch path (`_pg_dispatch`, referenced around `api.py:3548`) emits equivalent pub/sub events. Nothing in the observations shows pubsub calls from the PG backend.
- Deep-copy semantics: `ChangeEvent` values are strings/`None` (immutable), so freezing the dataclass is sufficient; no defensive copying of `node_id` is needed.

## Topics to Explore

- [function] `reasons/api.py:_with_network` — The context manager that captures before/after snapshots, calls `save`, and gates on `pubsub.has_subscribers` (lines 139–158); this is where the entire emission contract is enforced.
- [file] `reasons/pubsub.py` — The single source of truth for subscription state, fan-out, and `compute_changes`; verify there is no `__all__` and confirm exactly what the shim re-exports.
- [file] `tests/test_pubsub.py` — The executable contract: 347 lines of behaviour assertions you should treat as the spec for any consumer.
- [general] `pg-backend-pubsub` — Investigate whether `_pg_dispatch` / `api.topics(db_path=..., pg_conninfo=...)` emit analogous events, or whether pub/sub is file-DB-only.
- [file] `reasons_lib/pubsub.py` — The 3-line compatibility shim; verify downstream importers and whether any rely on names not in the intended surface (e.g. `logger`).

## Beliefs

- `pubsub-channel-key-is-db-path-or-global` — The only channel discriminator is `db_path`; `None` is the global channel that receives events for every database.
- `pubsub-publish-skips-empty-batches` — `publish` returns immediately when `events` is empty, so subscribers are never called with `[]`.
- `pubsub-subscriber-errors-do-not-propagate` — Each subscriber is invoked inside `try/except Exception` that logs and swallows, so one failing subscriber cannot abort the save or starve others.
- `change-event-has-no-cause-metadata` — `ChangeEvent` carries only `type`, `node_id`, `old_truth_value`, `new_truth_value`, `db_path`; cause and cascade-root information is explicitly absent by design.
- `pubsub-events-emitted-after-save` — `pubsub.publish` is called after the write transaction commits, so any callback that re-queries the DB sees the persisted truth values (`api.py:158`, `test_events_after_save`).
- `topics-module-is-unrelated-to-pubsub` — `api.topics` and `tests/test_topics.py` perform word-frequency topic extraction over node IDs and involve no pub/sub channels or subscriptions.
