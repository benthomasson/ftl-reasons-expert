---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: How `compute_changes(before, after, db_path)` turns a full truth-value snapshot diff into events, and why the snapshot only happens when there are subscribers on the write path.

`compute_changes` is a pure in-memory diff function defined at `reasons/pubsub.py:89`. It does **not** query SQLite or inspect the live network; it only compares two dictionaries mapping `node_id -> truth_value` and converts the differences into `ChangeEvent` objects.

## How the diff becomes events

The function makes two passes:

1. **Modified and added nodes** — iterate `after.items()` (`reasons/pubsub.py:98`):
   - Look up the previous value with `before.get(nid)`.
   - If `old_tv != new_tv`, emit a `ChangeEvent`.
   - A changed node gets `old_truth_value=old_tv`, `new_truth_value=new_tv`.
   - An added node naturally appears as `old_truth_value=None`, because `before.get(nid)` returns `None`.
   - The event type is always `"node_changed"` and the supplied `db_path` is stamped onto the event.

2. **Removed nodes** — iterate keys in `before` (`reasons/pubsub.py:107`):
   - If a node id is no longer present in `after`, emit a `ChangeEvent`.
   - It carries the old truth value and `new_truth_value=None`.

The event shape is defined by `ChangeEvent` at `reasons/pubsub.py:19`. The docstring notes that events carry **what changed** but not **why**, because the write-path hook only sees before/after snapshots and does not know cause or cascade root (`reasons/pubsub.py:21-26`).

Example conceptually:

```python
before = {"a": "IN", "b": "OUT"}
after  = {"a": "OUT", "c": "IN"}
```

`compute_changes` produces:

- `a`: `IN -> OUT`
- `c`: `None -> IN`
- `b`: `OUT -> None`

All three are `type="node_changed"` events.

## Why the snapshot only happens when there are subscribers

The write path gates snapshot work with:

```python
if write and pubsub.has_subscribers(db_path):
```

at `reasons/api.py:139`.

`has_subscribers` is defined at `reasons/pubsub.py:61`. It returns `True` if either:

- there is a global subscriber under the `None` key, or
- there is at least one subscriber registered for the specific `db_path`.

The pub/sub system is in-process and ephemeral: `_subscribers` is just a module-level dict (`reasons/pubsub.py:35`), and `publish` dispatches events to currently registered callbacks (`reasons/pubsub.py:69-86`). If no subscriber exists at write time, computing a full truth-value snapshot and diff would produce events with nowhere to go. So the snapshot is skipped as an optimization: no listeners means no need to build a before/after diff.

When the guard is true, the write path later calls:

```python
events = pubsub.compute_changes(...)
```

at `reasons/api.py:155`. The resulting events are then eligible for dispatch via `publish`, which sends each batch to global subscribers plus subscribers registered for that `db_path` (`reasons/pubsub.py:79-80`). Subscriber exceptions are logged and swallowed (`reasons/pubsub.py:82-86`).

The module docstring explains the timing: subscribers receive batched events after each `_with_network` write transaction completes, so events reflect the final stable state rather than intermediate cascade states (`reasons/pubsub.py:1-7`).

One caveat: the provided observations show the guard at `reasons/api.py:139` and the `compute_changes` call at `reasons/api.py:155`, but not the full lines between them. The exact construction of the `before`/`after` maps and the precise `publish` call site are therefore inferred from the surrounding API and pubsub module, not fully visible in the excerpt.

## Topics to Explore

- [function] `reasons/api.py:_with_network` — Contains the write-path guard and `compute_changes` call; trace exactly how `before`/`after` snapshots are built and when events are published.
- [function] `reasons/pubsub.py:publish` — Shows how event batches are routed to global and db-specific subscribers and how subscriber failures are contained.
- [function] `reasons/pubsub.py:has_subscribers` — Defines the predicate that gates write-path snapshotting.
- [file] `tests/test_pubsub.py` — Covers `compute_changes` behavior and subscribe/unsubscribe semantics; useful for confirming event ordering and deletion handling.
- [general] `write-path-snapshot-lifecycle` — Verify whether the `after` snapshot is taken after `Storage.save` and whether `publish` runs only after the transaction commits.

## Beliefs

- `compute-changes-is-pure-diff` — `compute_changes` compares in-memory `before`/`after` dictionaries and does not read or write SQLite.
- `compute-changes-emits-node-changed-for-all-diffs` — Modifications, additions, and removals are all emitted as `ChangeEvent(type="node_changed")`; removals use `new_truth_value=None`.
- `snapshot-requires-subscribers` — The write-path snapshot is skipped unless `write` is true and `pubsub.has_subscribers(db_path)` is true.
- `has-subscribers-checks-global-and-db-specific` — `has_subscribers` returns true when either the global `None` subscriber list or the `db_path`-specific subscriber list is non-empty.
- `publish-routes-to-global-and-db-specific` — `publish` sends each event batch to global subscribers plus subscribers registered for the event’s `db_path`.
