---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: The caller that triggers `publish` on `__exit__` after save; where before/after snapshots are captured.

# Where `publish` fires and how before/after snapshots are computed

## The short version

`pubsub.publish(...)` is invoked from the `__exit__` method of a `_with_network`-style context manager defined in `reasons/api.py`. That `__exit__` is the only place in the codebase that calls `pubsub.publish` (line 158), and it runs **after** `Storage.save()` has flushed the network to SQLite. The before/after data used to build the event list are plain `dict[str, str]` truth-value maps that `__exit__` passes to `pubsub.compute_changes`, which diffs them into `ChangeEvent`s.

## Tracing the call

1. **The caller: `__exit__` in `reasons/api.py`**
   - Definition: `reasons/api.py:146` → `def __exit__(self, exc_type, exc_val, exc_tb):`
   - Publish site: `reasons/api.py:158` → `pubsub.publish(events, db_path)`
   - `reasons/pubsub.py:70`'s docstring confirms the caller: *"Called by `_with_network.__exit__` after save completes."*

2. **Inside `publish` (`reasons/pubsub.py:70`)**
   - It's a no-op if `events` is empty (`if not events: return`).
   - It fans the list out to every subscriber registered globally (`db_path=None`) plus every subscriber scoped to the specific database being written.
   - Subscriber exceptions are caught and logged (`logger.exception(...)`), so a bad subscriber cannot roll back or fail the write.

3. **Inside `Storage.save` (`reasons/storage.py`, class `Storage`, method `save`)**
   - The persistence step runs between the two snapshots. It opens a `with self.conn:` transaction, snapshots externally-added tags/sources, deletes/rewrites `nodes`, `justifications`, `nogoods`, `node_tags`, `node_sources`, `repos`, `propagation_log`, `network_meta`, and rebuilds the FTS index.
   - Because `save` commits atomically, the "after" truth-value map read by `__exit__` reflects a stable, post-commit network — not an intermediate cascade state. This is the invariant the pubsub module docstring states: *"Events are never emitted for intermediate cascade states — only the final stable state is visible."*

## Where the before/after snapshots are captured

The diff is produced by `compute_changes(before, after, db_path)` in `reasons/pubsub.py` (signature around line 91):

```python
def compute_changes(
    before: dict[str, str],
    after: dict[str, str],
    db_path: str,
) -> list[ChangeEvent]:
```

- Both arguments are **flat dicts mapping `node_id → truth_value`** (`'IN'`, `'OUT'`, etc.).
- `compute_changes` walks `after` to find nodes whose value changed, and `before` to find nodes that disappeared, producing one `ChangeEvent` per difference. It emits **only** `old_truth_value`/`new_truth_value` — no cause string, no `cascade_root`, no operation name.
- The "why" metadata is deliberately out of scope: `ChangeEvent`'s docstring (`reasons/pubsub.py:16-25`) and the module docstring both state the hook *"only sees before/after snapshots, not which operation triggered the change."*

## What is inferable but **not** directly observed

The observations confirm the *call site* (`api.py:158`) and the *diff function* (`pubsub.compute_changes`), but they do **not** show the exact lines where the `before` and `after` dicts are built. Those lines live in `_with_network.__enter__` (which must build the "before" snapshot from `network.nodes`) and in the body of `__exit__` between lines 146 and 158 (which must build the "after" snapshot from the same `network.nodes` after `storage.save` returns). To pin those down you need the full body of `_with_network` in `reasons/api.py` — specifically `__enter__` and the lines immediately preceding line 158.

## Topics to Explore

- [file] `reasons/api.py` — Read the full `_with_network` context manager (around lines 120–170) to see `__enter__`, the `storage.save()` call, and the snapshot dict construction that feed `compute_changes`.
- [function] `reasons/pubsub.py:compute_changes` — The exact diff algorithm that produces `ChangeEvent`s; note it only emits `node_changed` and only for truth-value deltas.
- [function] `reasons/storage.py:save` — File-level atomic persistence step; how the `with self.conn:` block guarantees the "after" snapshot is post-commit.
- [general] `pubsub-subscriber-contract` — How subscribers are registered via `subscribe(callback, db_path=...)` and why subscriber exceptions are swallowed.
- [file] `tests/test_pubsub.py` — Tests around lines 94, 101, 106, 124 exercise `publish` with various event shapes; useful for confirming the event contract.

## Beliefs

- `publish-caller-is-with-network-exit` — `pubsub.publish` is called from exactly one production site: `_with_network.__exit__` at `reasons/api.py:158`.
- `publish-runs-after-storage-save` — The publish call executes after `Storage.save()` has committed the SQLite transaction, so subscribers see post-commit state.
- `snapshots-are-node-id-to-truth-value-maps` — Both `before` and `after` passed to `compute_changes` are `dict[str, str]` keyed by node id.
- `change-events-lack-causation` — `ChangeEvent` carries only `old_truth_value`/`new_truth_value`/`node_id`/`db_path`; it does not carry cause or `cascade_root`.
- `subscriber-errors-are-isolated` — `publish` wraps each subscriber call in `try/except Exception` and logs via `logger.exception`, so a failing subscriber cannot break the write path.
- `empty-event-batches-are-dropped` — `publish` returns immediately when `events` is empty, so subscribers are not invoked for no-op transactions.
