---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Investigate whether `_pg_dispatch` / `api.topics(db_path=..., pg_conninfo=...)` emit analogous events, or whether pub/sub is file-DB-only.

# Investigation: Do PG-backed writes emit pub/sub events?

**Short answer: No. Pub/sub is file-DB-only.** The PostgreSQL path through `_pg_dispatch`/`PgApi` bypasses every hook that the file-DB path uses to publish `ChangeEvent`s, and the event schema itself is anchored to a `db_path` string with no PG/domain analog.

## Evidence from the code

### 1. `_pg_dispatch` is a pure passthrough (`reasons/api.py:184`)

```python
def _pg_dispatch(pg_conninfo, domain_id, method_name, **kwargs):
    """Dispatch a call to the PostgreSQL backend."""
    from .pg import PgApi
    with PgApi(pg_conninfo, domain_id) as pg:
        return getattr(pg, method_name)(**kwargs)
```

No `pubsub.has_subscribers`, no snapshot, no `pubsub.compute_changes`, no `pubsub.publish`. It just opens a `PgApi` context and calls one method on it. Every PG call site (all 30+ usages listed, from `init_db` at line 204 to `import_beliefs` at 1916) goes through this same function, so none of them can emit events.

### 2. The *only* pub/sub integration point is `_with_network.__exit__` (reasons/api.py, ~130–190)

```python
def __exit__(self, exc_type, ...):
    if write and exc_type is None:
        if self.store:
            self.store.save(self.network)
            if self._before is not None:
                after = {nid: n.truth_value for nid, n in self.network.nodes.items()}
                events = pubsub.compute_changes(self._before, after, db_path)
                pubsub.publish(events, db_path)
    ...
```

Every public API function in `reasons/api.py` satisfies:

```python
if pg_conninfo:
    return _pg_dispatch(...)          # <-- exits early, never reaches _with_network
with _with_network(db_path, write=...) as net:
    ...
```

So the PG branch never enters `_with_network` and therefore never runs the publish logic. Additionally, the snapshot is guarded by `write=True` **and** `self.store` being non-None — both of which are false in the PG path.

### 3. `PgApi` itself has no pubsub hooks (reasons/pg.py:117–132)

```python
def close(self):
    if self._owns_conn:
        self.conn.close()

def __enter__(self): return self

def __exit__(self, exc_type, *exc):
    if exc_type is not None:
        self.conn.rollback()
    self.close()
```

Write methods like `add_node` (pg.py:255), `retract_node` (~pg.py:330), `add_justification` (pg.py:337), `assert_node` (~pg.py:380) commit their own transactions internally — but none of them call into `reasons.pubsub`. There is no equivalent of the file-DB's before/after diff.

### 4. The `ChangeEvent` schema is path-shaped, not connection-shaped (reasons/pubsub.py:19–25)

```python
@dataclass(frozen=True)
class ChangeEvent:
    type: str
    node_id: str
    old_truth_value: str | None
    new_truth_value: str | None
    db_path: str              # required; no pg_conninfo / domain_id field
```

And the registry key is `_subscribers: dict[str | None, list[Subscriber]]` with `subscribe(callback, db_path=None)` — either "all" or one specific file path. There is no natural key for `(pg_conninfo, domain_id)`; a hypothetical PG event would need either a synthetic `db_path` (e.g. `"pg://…/proj-1"`) or a schema change.

### 5. Tests confirm this gap

- `tests/test_pubsub.py` — every integration test uses `db_path=db` (SQLite). No PG test exists.
- `tests/test_pg_dispatch.py` — asserts dispatch routing and kwargs; **never** asserts pubsub behavior. `_pg_dispatch` tests use `MagicMock` for `PgApi` and check the called method, not events.

So there is no test guaranteeing PG writes emit events, which is consistent with them not emitting events.

## What about `api.topics` specifically?

Two definitions were found:

- `reasons/api.py:3538` — `def topics(`
- `reasons/mcp_server.py:583` — `def topics(limit: int = 20) -> str:` (this is the MCP tool wrapper, not the data-layer function).

**The body of `reasons/api.py:topics` is not included in the observations**, only the signature line. I cannot confirm from the provided snippets whether it:

1. Raises `NotImplementedError` for `pg_conninfo` (as `add_tags`/`remove_tags`/`get_tags` do at api.py:1079, 1106, 1131), or
2. Dispatches to a `PgApi.topics` method.

Either way, note two things:

- `topics` is a **read** operation (in the file-DB path it would query `node_tags` for `topic:*` values — cf. `_filter_by_tags` and `_get_node_tags` at api.py:57/95). `_with_network` only publishes events when `write=True`, so even a SQLite `topics` call emits nothing.
- If `topics` does dispatch via `_pg_dispatch`, it emits nothing for the same structural reason described above — `_pg_dispatch` doesn't touch pubsub at all.

So the answer for `topics` is doubly negative: it is read-only, and the PG path has no event plumbing either way.

## Conclusion

- **File-DB write path** → `_with_network.__exit__` → `pubsub.compute_changes` + `pubsub.publish` → subscribers.
- **PG write path** → `_pg_dispatch` → `PgApi.<method>` → COMMIT. No event snapshot, no diff, no publish.
- **Read paths** (either backend) emit nothing because publishing is gated on `write=True`.
- The `ChangeEvent` dataclass and the subscriber registry are both keyed by `db_path`, structurally tying pub/sub to the SQLite storage model.

Any consumer that relies on `pubsub.subscribe(...)` to react to truth-value changes will **silently miss all PostgreSQL-backed mutations**. Bridging the gap would require adding a publish hook either inside `PgApi`'s write methods (needs a before/after diff inside each transaction) or wrapping `_pg_dispatch` with the same snapshot logic `_with_network` uses — plus either a synthetic `db_path` key or a schema extension (e.g. `domain_id`/`conninfo` fields on `ChangeEvent`).

## Topics to Explore

- [file] `reasons/api.py` — Read the full body of `topics` at line 3538 to confirm whether it raises `NotImplementedError` for `pg_conninfo` (like `get_tags`/`add_tags`) or dispatches to `PgApi`.
- [function] `reasons/pg.py:PgApi.add_node` — Trace the commit points inside a PG write to see where a `pubsub.publish` hook would have to be inserted (or where a snapshot would need to live).
- [file] `reasons/pubsub.py` — Evaluate the `ChangeEvent` schema and `_subscribers` key design to understand what a PG-aware event would need to change.
- [function] `reasons/api.py:_with_network` — Confirm the write-gating (`if write and exc_type is None`) that ensures even read-only SQLite calls never publish, so you understand the "no events for reads" invariant.
- [general] `pg-pubsub-parity` — Decide whether the product requirement is to bridge PG→pubsub (adding hooks) or to document/formalize that pubsub is a SQLite-only feature.

## Beliefs

- `pg-dispatch-bypasses-pubsub` — `_pg_dispatch` (reasons/api.py:184) returns immediately from `PgApi.<method>` without ever calling `pubsub.compute_changes` or `pubsub.publish`, so all 30+ `pg_conninfo`-branched API calls emit zero `ChangeEvent`s.
- `with-network-is-only-publish-site` — The sole call sites of `pubsub.publish`/`pubsub.compute_changes` are inside `_with_network.__exit__` (reasons/api.py), which is entered only when `pg_conninfo` is falsy.
- `pubsub-publish-requires-write-true` — Events are published only when `write=True` **and** the SQLite `Storage` object exists, so read-only calls like `topics`/`get_status` never emit events on either backend.
- `change-event-has-db-path-only` — `ChangeEvent.db_path: str` (reasons/pubsub.py:22) is the sole location field; there is no `pg_conninfo`/`domain_id` field, so subscribers are partitioned by filesystem path.
- `pg-api-exit-has-no-pubsub-hook` — `PgApi.__exit__` (reasons/pg.py) only performs rollback on exception and `close()` on success; it contains no event-diff or publish logic.
