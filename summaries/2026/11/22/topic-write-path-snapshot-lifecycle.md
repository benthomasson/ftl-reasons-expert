---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Verify whether the `after` snapshot is taken after `Storage.save` and whether `publish` runs only after the transaction commits.

## Verification

Short answer: **The commit behavior of `Storage.save` is visible, but the actual `after` snapshot/publish ordering is not fully verifiable from the provided observations.** The comments in `reasons/pubsub.py` state the intended ordering, but the `_with_network` implementation that would prove it is missing.

### 1. Does `Storage.save` commit before returning?

Yes, from the shown code.

- `Storage` is defined at `reasons/storage.py:123`.
- `Storage.save` is defined at `reasons/storage.py:206`.
- The entire write body of `save` is wrapped in:

```python
with self.conn:
    ...
```

For `sqlite3.Connection`, the connection context manager commits on normal block exit and rolls back on exception. Since `save` has no explicit `conn.commit()` and all writes are inside `with self.conn:`, the transaction commits as `save` returns.

So if `_with_network` calls `storage.save(network)` and only then takes the `after` snapshot, that snapshot is taken after the save transaction has committed.

### 2. Is the `after` snapshot taken after `Storage.save`?

**Not directly verifiable from the observations.**

`after` appears conceptually in `reasons/pubsub.py`, especially in `compute_changes(before, after, db_path)`, but the code that constructs `after` is not shown. It is likely inside `_with_network.__exit__`, but that implementation is missing.

The `reasons/pubsub.py` module docstring says:

> Subscribers receive batched events after each `_with_network` write transaction completes (after save).

And the `publish` docstring at `reasons/pubsub.py:70` says:

> Called by `_with_network.__exit__` after save completes.

That is strong evidence of intended behavior, but it is still documentation, not the executable call site.

### 3. Does `publish` run only after the transaction commits?

**Also not directly verifiable from the shown code.**

- `publish` is defined at `reasons/pubsub.py:70`.
- `publish` itself contains no transaction or commit logic; it only dispatches `ChangeEvent`s to subscribers.
- `Storage.save` contains no call to `publish`; the search result `find_commit_after_publish_in_storage` is empty.
- Therefore, the ordering depends entirely on the missing `_with_network.__exit__` implementation.

If `_with_network.__exit__` does:

```python
self.storage.save(network)
after = snapshot(network)
publish(compute_changes(before, after, db_path), db_path)
```

then yes: `Storage.save` commits first, the `after` snapshot is taken after save, and `publish` runs after the save transaction commits.

But if `publish` is called inside a larger transaction, or before `save` returns, that would not hold. The provided observations do not show the call site.

### Caveat about the search results

`find_transaction_commit_patterns` returned no matches, but the shown `Storage` implementation clearly contains `with self.conn:` in `save` and `self.conn.commit()` in `_init_schema`. Those search results appear incomplete or incorrectly scoped. The implementation text is more reliable here.

## Topics to Explore

- [general] `_with_network-exit` — Locate the `_with_network` context manager implementation to verify exactly where the `before`/`after` snapshots are taken and where `publish` is called.
- [function] `reasons/pubsub.py:publish` — Find all call sites of `publish` and confirm they occur after `Storage.save` returns.
- [function] `reasons/storage.py:save` — Inspect the `with self.conn:` commit/rollback behavior and whether any caller wraps `save` in an outer transaction.
- [general] `pubsub-ordering-tests` — Search for tests that assert publish ordering relative to save/commit; the provided search found no such tests.

## Beliefs

- `storage-save-commits-on-context-exit` — `Storage.save` at `reasons/storage.py:206` writes inside `with self.conn:`, so SQLite commits when the block exits normally and rolls back on exception.
- `storage-save-does-not-publish` — `Storage.save` contains no call to `publish`; event dispatching is handled elsewhere.
- `publish-is-dispatch-only` — `publish` at `reasons/pubsub.py:70` only fans out `ChangeEvent`s to subscribers and contains no transaction/commit logic.
- `pubsub-docstring-asserts-post-save-publish` — `reasons/pubsub.py` states that publish is called by `_with_network.__exit__` after save completes, but that call site is not shown.
- `with-network-exit-is-missing` — The `_with_network` implementation needed to prove `after` snapshot and publish ordering is absent from the observations.
