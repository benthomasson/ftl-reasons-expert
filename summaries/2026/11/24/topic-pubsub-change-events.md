---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: How `_with_network` diffs pre/post truth values and publishes events; relevant to any write-side side effect

## How `_with_network` diffs truth values and publishes events

The observed code shows the contract clearly, even though the full `_with_network` body is not included. The definition is at `reasons/api.py:116`:

```python
def _with_network(db_path: str, write: bool = False):
```

The pubsub module documents that `publish()` is called by `_with_network.__exit__` after `save` completes, and that subscribers only see the final stable state after a write transaction.

### Write-side sequence

For a write context such as `reasons/api.py:230` or `reasons/api.py:336`:

```python
with _with_network(db_path, write=True) as net:
    ...
```

the intended write-side flow is:

1. `_with_network` captures the pre-write truth-value snapshot, likely `node_id -> truth_value`.
2. The `with` body mutates the `Network`.
3. On successful exit with `write=True`, the network is saved through `reasons/storage.py:Storage.save`.
4. `_with_network` captures the post-save truth-value snapshot.
5. It calls `reasons/pubsub.py:compute_changes(before, after, db_path)`.
6. It calls `reasons/pubsub.py:publish(events, db_path)`.

This order matters: `publish` is documented as being called by `_with_network.__exit__` after save completes. Subscribers therefore see committed/final truth values, not intermediate propagation cascade states.

Read-only usages such as `reasons/api.py:253`, `:527`, `:594`, and `:662` use `write=False` or omit `write`; those are not expected to persist or publish truth-change events. The dry-run usage at `reasons/api.py:1562` uses:

```python
with _with_network(db_path, write=not dry_run) as net:
```

so a dry run suppresses the write-side save/publish path.

### How the diff works

`reasons/pubsub.py:compute_changes` accepts two mappings:

```python
before: dict[str, str]
after: dict[str, str]
```

It emits one `ChangeEvent` per changed node:

- If a node exists in `after` and its `new_tv` differs from `before.get(nid)`, it emits a `node_changed` event with the old and new truth value.
- If a node exists in `before` but not in `after`, it emits a `node_changed` event with `old_truth_value=before[nid]` and `new_truth_value=None`. That represents deletion/removal from the truth snapshot.
- If a node is unchanged, no event is emitted.
- If a node is newly added, `before.get(nid)` is `None`, so the event has `old_truth_value=None`.

The event shape is defined by `ChangeEvent` in `reasons/pubsub.py`:

```python
type: str
node_id: str
old_truth_value: str | None
new_truth_value: str | None
db_path: str
```

In `compute_changes`, `type` is always set to `"node_changed"`. The event does not carry `cause`, `cascade_root`, or operation metadata because `_with_network` only sees before/after snapshots.

### How publishing works

`reasons/pubsub.py:publish(events, db_path)`:

1. Returns immediately if `events` is empty.
2. Collects global subscribers registered under `None`.
3. Adds subscribers registered for the specific `db_path`.
4. Calls each subscriber with the whole `list[ChangeEvent]`.
5. Catches subscriber exceptions, logs them via `logger.exception`, and does not propagate them.

So a subscriber failure does not break the caller or roll back the write. Also, because `publish` returns early for an empty event list, a write transaction that changes no truth values produces no subscriber callback.

### Write-side implications

Any code path entering `with _with_network(db_path, write=True)` can trigger these side effects if the body changes truth values:

- persistence via `Storage.save`
- batched post-save pubsub delivery
- no per-intermediate-state events
- no events for unchanged truth values
- no cause/cascade metadata in the event

The exact body of `_with_network` is missing from the observations, so the precise snapshot variable names and exception behavior are not visible. In particular, it is not shown whether `_with_network` always builds snapshots or uses `has_subscribers()` to skip diffing when nobody is listening.

## Topics to Explore

- [function] `reasons/api.py:_with_network` — Inspect the exact before/after snapshot logic, save call, publish call, and exception handling on context exit.
- [function] `reasons/storage.py:Storage.save` — Confirm the save transaction boundary and whether pubsub publication occurs strictly after commit.
- [function] `reasons/pubsub.py:publish` — Review global versus per-database subscriber dispatch and the exception-swallowing behavior.
- [general] `write-vs-read-only-contexts` — Compare `write=True` call sites at `reasons/api.py:230`, `:336`, and `:1562` against `write=False` sites at `:253`, `:527`, and `:594`.
- [general] `subscriber-concurrency-and-lifecycle` — The `_subscribers` registry is module-global; check whether callers rely on thread safety, cleanup, or test isolation.

## Beliefs

- `_with_network-write-true-enables-pubsub-side-effects` — A `_with_network(db_path, write=True)` context is the path that can save the network and publish `ChangeEvent`s after exit.
- `compute_changes-emits-node-changed-for-truth-diffs` — `reasons/pubsub.py:compute_changes` emits one `ChangeEvent(type="node_changed")` for each node whose truth value changes or disappears between snapshots.
- `publish-requires-events` — `reasons/pubsub.py:publish` returns immediately for an empty event list and does not notify subscribers.
- `publish-occurs-after-save` — Pubsub is documented as called by `_with_network.__exit__` after save completes, so subscribers see final persisted truth values rather than intermediate cascade states.
- `subscriber-exceptions-do-not-propagate` — `publish` catches each subscriber exception and logs it instead of raising to the write caller.
