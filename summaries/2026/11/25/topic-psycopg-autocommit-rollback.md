---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Verify why passing an `autocommit=True` connection would silently persist the simulation, and how `PgApi.__init__` should be adjusted to defend against it.

## Why `autocommit=True` Silently Persists the Simulation

### How `PgApi` transaction semantics currently work

`PgApi.__init__` (`reasons/pg.py:101`) has **two branches** with asymmetric transaction handling:

```python
def __init__(self, conninfo, domain_id):
    _require_psycopg()
    if isinstance(conninfo, str):
        self.conn = psycopg.connect(conninfo, autocommit=False)   # line 107
        self._owns_conn = True
    else:
        self.conn = conninfo                                       # line 110
        self._owns_conn = False
    self.domain_id = str(domain_id)
```

- **String conninfo** → `psycopg.connect(..., autocommit=False)`. Every statement stays inside an explicit transaction until the method calls `self.conn.commit()` (e.g., `init_db`, around line 215 of the read output) or the connection is closed/rolled back.
- **Borrowed connection** (any non-string, e.g., a `psycopg.Connection`) → simply `self.conn = conninfo` with **no verification of `autocommit`**. The caller's state is inherited blind.

The "simulation" semantics depend on three invariants:

1. **`autocommit` is `False`** — statements stay uncommitted until explicitly committed.
2. **Explicit `commit()` only on success** — helper methods like `_add_node_raw` (`reasons/pg.py:180`, docstring at 181–184) explicitly say: *"Does NOT commit … Caller is responsible for transaction management."*
3. **`__exit__` calls `rollback()` on exception** (`reasons/pg.py:118`) or the connection is closed without commit (psycopg v3 rolls back pending work on close), which discards the pending transaction.

### Why an `autocommit=True` connection defeats this

When the caller passes a borrowed connection created with `autocommit=True`:

1. `self.conn = conninfo` — the branch takes the non-string path and never checks or forces `autocommit`.
2. In psycopg v3, every `cur.execute(...)` under `autocommit=True` is **immediately committed to the database** — there is no enclosing transaction.
3. Consequently:
   - `self.conn.rollback()` in `__exit__` becomes a **no-op** (nothing to roll back).
   - `self.conn.commit()` (e.g., `init_db`) becomes a **no-op** (already durable).
   - The "don't commit = simulation" contract (`_add_node_raw`) is silently violated — there is no uncommitted buffer to throw away.
4. Nothing in the code raises, warns, or detects the mismatch. The caller's intent to *simulate and roll back* is silently converted into *persist every mutation as it happens*.

Net effect: the moment a method runs a mutation (e.g., `add_node` via `_add_node_raw` at `reasons/pg.py:196`), the write is durable. Any subsequent exception, disconnect, or explicit rollback request cannot undo it, because that history was already committed at statement level.

### Recommended defensive change to `PgApi.__init__`

The safest fix is to **reject** a borrowed connection whose `autocommit` is not `False` (or to forcibly flip it). Rejecting is preferable because silently mutating the caller's connection state can surprise code that legitimately wanted autocommit for other work:

```python
def __init__(self, conninfo, domain_id):
    _require_psycopg()
    if isinstance(conninfo, str):
        self.conn = psycopg.connect(conninfo, autocommit=False)
        self._owns_conn = True
    else:
        # Defend against silently persisting "simulation" work
        if getattr(conninfo, "autocommit", False):
            raise ValueError(
                "PgApi requires a non-autocommit connection so that "
                "uncommitted simulation changes can be rolled back. "
                "Pass a conninfo string, or a psycopg connection with "
                "autocommit=False."
            )
        if getattr(conninfo, "closed", False):
            raise ValueError("PgApi received a closed connection.")
        self.conn = conninfo
        self._owns_conn = False
    self.domain_id = str(domain_id)
```

Rationale for the shape of the fix:

- **`getattr(..., "autocommit", False)`** keeps the check compatible with test doubles / `MagicMock` instances in `tests/test_pg_dispatch.py` (which mock `PgApi` at the class level, but any real double passed in will not have `autocommit=True` by default).
- **`raise ValueError`** is preferred over `conninfo.autocommit = False` because it (a) surfaces the bug at construction time instead of after the first silent commit, and (b) leaves the caller's connection in a state they own. If you *do* want to support auto-forcing, expose it as an opt-in flag (`PgApi(conn, domain_id, force_transactional=True)`), not the default.
- The same discipline should be applied inside `__exit__`: the decode-and-discard path is currently *"if exception: rollback; then close"* (`reasons/pg.py:117–120`). For borrowed connections, `close()` is a no-op (`_owns_conn=False`), so:
  - On a **clean exit with pending mutations**, `PgApi` leaves the transaction open for the caller to decide — fine only if the caller's connection is genuinely non-autocommit.
  - On an **autocommit connection**, that "open transaction to be decided later" concept doesn't exist — which is exactly the failure mode above.

### Caveats / what's missing

- The 200-line read only shows `init_db` and `_add_node_raw`. We have not seen the top-level mutation methods (e.g., `add_node`, `retract_node`, `assert_node`, `add_justification`) that call `_add_node_raw`, nor any explicit `simulate()` API, so we can only infer the simulation contract from the `_add_node_raw` docstring ("Caller is responsible for transaction management") and the presence of `commit()`/`rollback()` around line 215.
- `find_simulation_or_transaction_control` reports **0 matches** even though `self.conn.commit()` and `self.conn.rollback()` appear in the truncated read. The search likely treated the pipe-separated string as a literal. Don't trust that "0" — the file definitely contains `commit`/`rollback` calls (visible at `reasons/pg.py:215` region and `reasons/pg.py:118`).
- There is no test in the visible slices that constructs `PgApi` with `autocommit=True`, so this bug is currently uncovered by the test suite. A regression test that asserts `ValueError` on an `autocommit=True` connection should accompany the fix.

## Topics to Explore

- [file] `reasons/pg.py` — Read the full 3,736-line module to enumerate every method that calls `self.conn.commit()` versus every method that deliberately defers to the caller; this reveals the exact blast radius of the `autocommit=True` bug.
- [function] `reasons/pg.py:__exit__` — Confirm the current behaviour on clean exit vs exception path, and decide whether borrowed-connection exits should `rollback()` or require the caller to be transactional.
- [function] `reasons/pg.py:_add_node_raw` — The docstring ("Caller is responsible for transaction management") is the clearest statement of the simulation contract; any top-level method relying on it inherits the same `autocommit` assumption.
- [file] `tests/test_pg.py` — Inspect the existing tests around line 538 (the `PgApi(os.environ["DATABASE_URL"], project2)` usage) to see whether borrowed connection paths are ever exercised.
- [general] `psycopg-v3-transaction-semantics` — Understand close-without-commit behavior in psycopg v3, which is what currently makes "simulation" work for owned connections but fails silently for borrowed `autocommit=True` connections.

## Beliefs

- `borrowed-connection-skips-autocommit-check` — `PgApi.__init__` (`reasons/pg.py:110`) assigns a non-string `conninfo` to `self.conn` without inspecting or enforcing `autocommit`.
- `autocommit-true-defeats-rollback` — A psycopg connection with `autocommit=True` commits each statement immediately, making `self.conn.rollback()` a no-op.
- `__exit__-rolls-back-only-on-exception` — `PgApi.__exit__` (`reasons/pg.py:117`) only calls `rollback()` when an exception is raised; clean exit leaves the transaction to the caller or lets `close()` roll it back.
- `owned-connection-uses-autocommit-false` — The string-conninfo branch in `__init__` explicitly passes `autocommit=False` (`reasons/pg.py:107`), which is the source of the transactional safety PgApi assumes.
- `add-node-raw-is-transaction-neutral` — `_add_node_raw` (`reasons/pg.py:180`) deliberately does not commit, per its docstring, so any persistent write on the borrowed-connection path is caused by the connection's `autocommit` setting, not by the method itself.
