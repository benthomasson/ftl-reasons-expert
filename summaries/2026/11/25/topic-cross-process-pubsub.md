---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Current design needs an external broker, DB notification mechanism, or polling layer to support cross-process subscribers.

## Why the current pub/sub is process-local only

`reasons/pubsub.py` is an in-process observer registry, not a message transport.

- The subscriber registry is a module-level dict: `_subscribers: dict[str | None, list[Subscriber]]` at `reasons/pubsub.py:35`.
- `subscribe()` at `reasons/pubsub.py:38` and `unsubscribe()` at `reasons/pubsub.py:50` only mutate that in-memory dict.
- `has_subscribers()` at `reasons/pubsub.py:61` checks only the current process’s dict.
- `publish()` at `reasons/pubsub.py:70` builds its target list from the current process’s `_subscribers` values, then invokes callbacks directly:

  ```python
  targets = list(_subscribers.get(None, []))
  targets.extend(_subscribers.get(db_path, []))
  ```

Because `_subscribers` is module-global state, Process A and Process B each have their own copy. If Process B calls `subscribe(...)`, Process A cannot see that callback. When Process A performs a write and calls `publish()`, it dispatches only to callbacks registered inside Process A. There is no shared broker, database queue, notification channel, or polling loop.

The API write path confirms this is tied to the writer process:

- `reasons/api.py:23` imports `pubsub`.
- `reasons/api.py:139` gates event snapshotting on `pubsub.has_subscribers(db_path)`.
- `reasons/api.py:155` calls `pubsub.compute_changes(...)`.
- `reasons/api.py:158` calls `pubsub.publish(events, db_path)`.

That `has_subscribers()` check is especially important: if the writer process has no local subscriber, it may not compute or publish events at all. So adding a broker only inside `publish()` would still not be enough unless the write path is changed to always emit to a durable transport/outbox or to detect remote subscribers. A reader process subscribing in its own interpreter does not affect the writer’s `has_subscribers()` result.

The event model is also local-only and non-durable. `compute_changes()` produces `ChangeEvent` records by diffing before/after truth-value snapshots. Events contain `node_id`, `old_truth_value`, `new_truth_value`, and `db_path`, but no event ID, sequence number, transaction ID, timestamp, acknowledgement, retry state, or replay store. Subscriber exceptions are caught and logged in `publish()` (`reasons/pubsub.py:70`), which is fine for an in-process observer hook but does not provide cross-process delivery semantics.

The storage/service layers do not fill the gap:

- `reasons/pg.py` has no event transport code. `grep_storage_for_event_mechanisms` returns zero matches for `subscribe|publish|notify|event|pubsub`.
- The PostgreSQL schema shown in `reasons/pg.py` defines `rms_nodes`, `rms_justifications`, `rms_nogoods`, `rms_propagation_log`, and `rms_network_meta`; it does not define an outbox/event table or `LISTEN/NOTIFY` triggers.
- `rms_propagation_log` is an audit-style table with `action`, `target`, and `value`; it is not a complete `ChangeEvent` stream with old/new truth values and transaction batching.
- `reasons/mind_service.py` provides HTTP `fetch_export()` and `push_belief()` helpers, but no subscription or event-stream endpoint.
- `grep_mcp_server_for_subscription_mechanisms` returns zero matches.

So to support cross-process subscribers, the design needs one of:

1. an external broker such as Redis/NATS/Kafka;
2. a database notification layer such as PostgreSQL `LISTEN/NOTIFY` plus triggers/outbox;
3. a polling layer over a durable event/outbox table or change log;
4. equivalent change-data-capture or file-watch mechanism for SQLite.

The current code has none of those. It only supports callbacks registered inside the same Python process that performs the write.

## Topics to Explore

- [function] `reasons/api.py:_with_network` — Shows exactly where the write transaction snapshots state and calls `pubsub.compute_changes()` / `pubsub.publish()`.
- [file] `reasons/pubsub.py` — The in-process registry and dispatcher that must be replaced or wrapped for cross-process delivery.
- [file] `reasons/pg.py` — PostgreSQL backend where a `LISTEN/NOTIFY`, trigger, or outbox implementation would likely live.
- [general] `cross-process-event-transport` — Compare broker, DB-notification, outbox, and polling approaches for delivery guarantees and operational cost.
- [file] `reasons/mind_service.py` — Existing HTTP client for export/push behavior, relevant if cross-process sync should use a remote service instead of a broker.

## Beliefs

- `pubsub-subscribers-are-process-local` — `_subscribers` in `reasons/pubsub.py` is a module-global dict and is not shared across processes.
- `pubsub-publish-targets-local-registry-only` — `publish()` dispatches only to callbacks found in the current process’s `_subscribers` dict.
- `api-skips-events-without-local-subscribers` — `reasons/api.py:139` gates `compute_changes()` and `publish()` on `pubsub.has_subscribers(db_path)`, so a writer with no local subscriber may emit no events at all.
- `pg-backend-has-no-event-transport` — `reasons/pg.py` contains no `LISTEN/NOTIFY`, trigger, outbox, or pub/sub publishing code.
- `change-events-are-final-snapshot-diffs` — `pubsub.compute_changes()` emits `ChangeEvent` records by diffing before/after truth-value snapshots and does not include cause or cascade-root metadata.
