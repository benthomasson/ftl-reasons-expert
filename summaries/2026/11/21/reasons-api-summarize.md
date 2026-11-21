---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Function: summarize in reasons/api.py

## `summarize` — `reasons/api.py`

A thin, high-level API function that creates a *summary node*: a belief that abstracts over a group of existing nodes. It is the public entry point ("standalone function any Python caller can use") for the summarization primitive — the actual logic lives in `Network.summarize` (SQLite) or `PgApi.summarize` (PostgreSQL).

### 1. Purpose

Provide a backend-agnostic Python entry point (usable from CLI, LangGraph tools, and scripts) for creating a summary node over a set of existing beliefs, without leaking the `Storage`/`PgApi` lifecycle into callers. Consistent with the module's overall contract: open the DB, do the work, save, close, and return a JSON-serializable `dict`.

### 2. Contract

**Preconditions (not enforced by the function itself):**
- On the SQLite path: `db_path` exists and contains a loadable network, or `_with_network`/`Storage` will fail.
- On the PostgreSQL path: `pg_conninfo` is a valid connection string and `domain_id` identifies the target domain.
- `over` contains node IDs that exist (or will be tolerated) in the network. `summary_id` must be a valid, non-colliding node identifier. **Neither is checked here — validation is delegated to `Network.summarize` / `PgApi.summarize`.**
- `text` is a non-empty belief statement (premise-like text).

**Postconditions:**
- Successful SQLite path: a node identified by `summary_id` describing `text` and abstracting `over` is created in the network, the network is saved to `db_path`, and a dict of the form `{"summary_id", "over", "truth_value"}` is returned (exact contents are whatever `Network.summarize` returns).
- Successful PG path: the equivalent mutation is made through `PgApi`, and its return value is returned.

**Invariants preserved by the wrapper:**
- Only one backend is exercised per call: if `pg_conninfo` is truthy, the SQLite code path is never touched (and vice-versa).
- All mutations go through `_with_network(..., write=True)`, so PubSub and the in-memory cache are kept coherent (see Side Effects).

### 3. Parameters

| Parameter | Type | Default | Meaning / edge cases |
|---|---|---|---|
| `summary_id` | `str` | — required | Identifier for the new summary node. Callers must ensure uniqueness; collisions surfaced by the backend, if at all. |
| `text` | `str` | — required | The summarization claim itself (read by downstream consumers as a belief). |
| `over` | `list[str]` | — required | Node IDs being summarized. Not validated for existence here. Passing an empty list is accepted by the type system; whether `Network.summarize` accepts it is up to the backend. |
| `source` | `str` | `""` | Free-form provenance string (typically `repo:path`). Empty means "no explicit source". Not validated or parsed. |
| `db_path` | `str` | `DEFAULT_DB = "reasons.db"` | Path to the SQLite DB. Ignored if `pg_conninfo` is set. |
| `pg_conninfo` | `str | None` | `None` | Dispatch flag. **Any truthy value** triggers the PostgreSQL path; falsy (including `""` and `None`) falls through to SQLite. |
| `domain_id` | `str | None` | `None` | Passed through to `PgApi(pg_conninfo, domain_id)` — ignored on SQLite path. |

Note: `over` happens to be a Python builtin-shadowing identifier used as a keyword further down (`over=over`), which is fine but a mild naming smell.

### 4. Return Value

- **SQLite path:** whatever `Network.summarize(summary_id, text, over, source=source)` returns. The docstring promises `{"summary_id": str, "over": list[str], "truth_value": str}`. Actual key set is determined by `Network`, not by this function.
- **PostgreSQL path:** whatever `PgApi.summarize(summary_id=..., text=..., over=..., source=...)` returns via `_pg_dispatch`. The `domain_id` is baked into the `PgApi` context.

Callers must handle:
- The missing keys if the backend deviates from the docstring (no schema enforcement).
- That `truth_value` will typically be `"IN"` (a freshly-created premise defaults IN), but the wrapper does not assert this.

### 5. Algorithm (step by step)

1. Enter the backend-branch on `pg_conninfo`:
   - If truthy: call `_pg_dispatch(pg_conninfo, domain_id, "summarize", summary_id=..., text=..., over=..., source=...)`.
     - `_pg_dispatch` imports `PgApi`, opens it as a context manager, and returns `getattr(pg, "summarize")(...)`.
     - Control returns immediately; the SQLite path is not entered.
2. Else (SQLite path):
   - `with _with_network(db_path, write=True) as net:` opens the context manager:
     - `mtime = os.path.getmtime(db_path)` — for cache keying.
     - Because `write=True`, the `_network_cache` is **bypassed**: a fresh `Storage(db_path)` is created and `store.load()` produces the network. (Read-only calls can hit the cache; writes always load fresh.)
     - In `__enter__`: if `write` and `pubsub.has_subscribers(db_path)`, snapshot the pre-change truth values into `self._before`.
   - Call `net.summarize(summary_id, text, over, source=source)`. Backend performs any validation, node creation, and recomputation; result dict is captured by `return`.
   - On context `__exit__`:
     - If no exception: `store.save(self.network)` persists the summary node.
     - If PubSub subscribers exist: compute before/after truth-value diffs and publish events.
     - `store.close()`.
     - Update `_network_cache[db_path] = (fresh_mtime, self.network)` (since the write succeeded).
3. Return the dict from `net.summarize`.

### 6. Side Effects

- **Filesystem write** on the SQLite path: `Storage.save()` rewrites the DB file at `db_path`.
- **In-memory cache mutation:** `_network_cache[db_path]` is updated to the post-write `(mtime, network)` pair.
- **PubSub notifications:** if any subscribers are registered for `db_path`, `pubsub.publish(events, db_path)` fires with the computed truth-value diff (creating a node adds an entry, so there is always a change to report on successful creation).
- **PostgreSQL path:** an external network/database mutation via `PgApi.summarize`; no local cache touches.
- No file handles leak: `Storage.close()` is always called in `__exit__`.

### 7. Error Handling

- **No error handling here.** The function is a pure pass-through:
  - `FileExistsError` / `OSError` from `Storage(db_path)` if the DB is missing or unreadable.
  - Whatever `Network.summarize` raises (e.g., `ValueError` for invalid `over`, `KeyError` for unknown node IDs, or none — depends entirely on `network.py`, which is not shown).
  - `_pg_dispatch` raises whatever `PgApi(...)` or `PgApi.summarize` raises (connection errors, `KeyError`, etc.).
- **Exceptions abort the write:** on any exception inside the context, `__exit__` sees `exc_type is not None` and **skips `store.save()`** — the DB remains unchanged. `store.close()` still runs, and the cache is *not* updated (the `not (write and exc_type is not None)` guard).
- **Silent failure modes to be aware of:**
  - Passing `pg_conninfo=""` is falsy → silently uses SQLite instead of erroring.
  - Passing `over=[]` is accepted at this layer — semantics depend on `Network.summarize`.
  - `summary_id` collisions are not detected by this function.

### 8. Usage Patterns

Typical caller idioms:

```python
# SQLite (default)
result = summarize(
    "summary-auth-flow",
    "Authentication is handled by the OAuth client in src/auth/",
    over=["auth-token-rotation", "auth-refresh", "auth-login"],
    source="docs:auth.md",
)
assert result["summary_id"] == "summary-auth-flow"

# PostgreSQL
summarize(
    "summary-auth-flow",
    "...",
    over=[...],
    pg_conninfo="postgresql://...",
    domain_id="acme",
)
```

Caller obligations:
- Ensure `over` IDs are valid in the target network (this function won't tell you until the backend does — or silently accepts them).
- Ensure `summary_id` does not collide with an existing node.
- Treat the returned dict as backend-dependent despite the documented three-key shape; prefer `result["summary_id"]` / `result["truth_value"]` and tolerate extra keys.
- Do not assume `over` ordering is preserved in the return value unless `Network.summarize` guarantees it.

### 9. Dependencies

- **`reasons.api._pg_dispatch`** — the PG backend shim; opens `PgApi(pg_conninfo, domain_id)` per call.
- **`reasons.api._with_network`** — context manager for load/operate/save with cache + PubSub semantics.
- **`reasons.storage.Storage`** — SQLite persistence (`load`, `save`, `close`).
- **`reasons.network.Network.summarize`** — actual summarization logic (behavior, return shape, validation all live there).
- **`reasons.pg.PgApi.summarize`** — PostgreSQL equivalent (imported lazily inside `_pg_dispatch`).
- **`reasons.pubsub`** — subscriber detection and change-event publication.
- **`reasons.api.DEFAULT_DB`** — default path constant, `"reasons.db"`.

## Topics to Explore

- [function] `reasons/network.py:Network.summarize` — The real implementation: how summary nodes are linked in the TMS (justifications, `over` metadata, truth-value computation, validation rules).
- [function] `reasons/api.py:_with_network` — The load/operate/save context manager; the caching, write-mode, and PubSub behavior that all write APIs silently rely on.
- [function] `reasons/api.py:_pg_dispatch` — The lazy dispatch shim that keeps PostgreSQL backend selection out of the hot module import path.
- [file] `tests/test_summarize.py` — Concretely pins the expected return shape, `over`-validation behavior, and any `KeyError`/`ValueError` expectations.
- [general] `pubsub-change-events` — How `compute_changes` detects diffs, and what consumers receive when a summary node appears.

## Beliefs

- `summarize-delegates-to-network-summarize` — On the SQLite path, all semantic and validation logic resides in `Network.summarize`; the API function is a pure wrapper.
- `summarize-uses-write-network-context` — Uses `_with_network(db_path, write=True)`, so a successful call persists the network and bypasses the read-only cache.
- `summarize-lacks-over-validation` — The function does not validate `over` node IDs, `summary_id` uniqueness, or emptiness of `text`.
- `summarize-publishes-pubsub-events` — The write context compares before/after truth values and calls `pubsub.publish` when subscribers are registered, even for a node creation.
- `summarize-dispatches-to-pg-when-pg-conninfo-set` — A truthy `pg_conninfo` routes the call through `_pg_dispatch` and never touches the SQLite path.
