---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/pubsub.py

# `reasons/pubsub.py` — In-Process TMS Change-Event Bus

## 1. Purpose

This module provides a **lightweight, in-process publish/subscribe channel** for a Truth Maintenance System (TMS). Its single responsibility: let external code observe *what* truth values changed after a write transaction commits, without coupling the TMS core to any specific consumer logic.

It sits at the boundary between the TMS write path (`_with_network` in `reasons/network.py`) and any downstream system that needs to react to belief changes (e.g., an MCP server pushing updates to a client, a background indexer, a test harness). The key design constraint stated in the docstring: **only the final stable post-save state is observable** — intermediate cascade states during the fixpoint computation are never leaked.

## 2. Key Components

| Symbol | Contract |
|--------|----------|
| `ChangeEvent` (frozen dataclass) | Immutable record: `type` (always `"node_changed"` today), `node_id`, `old_truth_value`, `new_truth_value`, `db_path`. No cause, no `cascade_root`. |
| `Subscriber` (type alias) | `Callable[[list[ChangeEvent]], None]` — receives a **batch**, not a single event. |
| `_subscribers` (module-level `dict`) | `dict[str | None, list[Subscriber]]`. Key `None` = global (all DBs); a concrete `db_path` = scoped to one database. |
| `subscribe(callback, db_path=None)` | Appends to the correct bucket. No duplicate check — calling twice registers the same callback twice. |
| `unsubscribe(callback, db_path=None)` | Removes first occurrence. Silently no-ops if not found. Cleans up the key if the list becomes empty. |
| `has_subscribers(db_path=None)` | Returns `True` if **any** subscriber exists globally *or* for the given `db_path`. Used as a fast-path guard before calling `compute_changes`. |
| `publish(events, db_path)` | Delivers to global subscribers **first**, then db-scoped subscribers. Catches and logs per-subscriber exceptions. |
| `compute_changes(before, after, db_path)` | Pure diff function. Returns a list of `ChangeEvent`. Detects value changes and deletions (node in `before` but absent from `after`). Does **not** detect pure insertions (node in `after` but not `before` with `old_tv=None` is still emitted as a change). |
| `clear()` | Wipes the entire `_subscribers` dict. For test teardown only. |

## 3. Patterns & Idioms

- **Observer / Pub-Sub**: Classic pattern, but intentionally minimal — no topics, no priorities, no async. One flat namespace per DB.
- **Module-level singleton state**: `_subscribers` lives at module scope, making it a process-wide registry. There is no class to instantiate; this is a "package-as-namespace" style common in small internal utilities.
- **Batch delivery**: Subscribers get `list[ChangeEvent]`, not individual events. This aligns with the "after the full transaction" semantic — one batch per write.
- **Frozen dataclass**: `ChangeEvent` is immutable after construction. Safe to share across threads without a lock (the fields are all read-only).
- **Error isolation**: `publish` wraps each subscriber call in `try/except Exception`. A misbehaving subscriber cannot block or crash other subscribers or the TMS itself.
- **Guard-fast-path**: `has_subscribers` lets the caller avoid the cost of `compute_changes` (a dict diff) when nobody is listening.

## 4. Dependencies

**Imports (this file's needs):**
- `logging` — standard library, used for error logging in `publish`.
- `dataclasses.dataclass` — for `ChangeEvent`.
- `typing.Callable` — for the `Subscriber` alias.

No third-party dependencies. No imports from other `reasons` modules. This is a deliberately isolated leaf.

**Imported by:**
- `reasons/api.py` — the main entry point that wires `publish` into the `_with_network` transaction hook.
- `reasons_lib/pubsub.py` — a parallel copy under the `reasons_lib` namespace (likely a vendored/shimmed version for a different import path).
- `tests/test_pubsub.py` — unit tests for the module itself.

## 5. Flow (Control & Data)

```
┌──────────────────────────────────────────────────────────────────┐
│  reasons/api.py  (or any _with_network consumer)                │
│                                                                  │
│  1.  has_subscribers(db_path)?  ──False──▶  skip (no-op)       │
│         │                                                        │
│       True                                                       │
│         ▼                                                        │
│  2.  compute_changes(before_snapshot, after_snapshot, db_path)  │
│         │  → list[ChangeEvent]  (possibly empty)                │
│         ▼                                                        │
│  3.  publish(events, db_path)                                   │
│         │                                                        │
│         ├──▶ global subscribers (key=None)                       │
│         │       for each: try subscriber(events)                 │
│         │                    except: logger.exception(...)       │
│         │                                                        │
│         └──▶ db-scoped subscribers (key=db_path)                 │
│                 for each: try subscriber(events)                 │
│                              except: logger.exception(...)       │
└──────────────────────────────────────────────────────────────────┘
```

`compute_changes` does two passes:
1. Iterate `after.items()` → if `before.get(nid) != new_tv`, emit a change (covers modifications **and** insertions where `old_tv` is `None`).
2. Iterate `before` keys → if `nid not in after`, emit a "deleted" event with `new_truth_value=None`.

## 6. Invariants

- **Atomicity of delivery**: A subscriber either receives the full batch or none of it. There is no partial delivery within a single `publish` call.
- **No intermediate states**: The caller contract (enforced in `reasons/api.py`, not here) guarantees that `before`/`after` are the pre-transaction and post-save snapshots. This module does not enforce it — it trusts the caller.
- **Order preservation**: Events in the returned list follow `after` dict insertion order, then `before`-only deletions. Subscribers can rely on this ordering.
- **Idempotent unsubscribe**: Calling `unsubscribe` with an unregistered callback is a silent no-op, not an error.
- **Key cleanup**: If removing a subscriber empties a list, the dict key is popped to avoid unbounded growth of empty lists.

## 7. Error Handling

| Location | Strategy |
|----------|----------|
| `publish()` — subscriber raises | Caught per-subscriber with `except Exception`. Logged via `logger.exception("pubsub subscriber raised an exception")`. **Does not propagate.** Remaining subscribers still receive the event. |
| `unsubscribe()` — callback not in list | `ValueError` from `list.remove` is caught and swallowed. No log, no re-raise. |
| `publish()` — empty event list | Early `return` before iterating any subscribers. Zero overhead. |
| `subscribe()` | No error path. Duplicates are allowed (intentional or oversight — no dedup check). |

There is **no** retry, no dead-letter queue, and no async delivery. This is synchronous, best-effort, fire-and-forget.

---

## Topics to Explore

- [function] `reasons/api.py:_with_network` — The transaction hook that calls `has_subscribers`, `compute_changes`, and `publish`. Understanding *when* and *with what snapshots* it fires is critical to reasoning about event semantics.
- [file] `tests/test_pubsub.py` — Reveals the expected edge cases: multi-DB scoping, global vs. scoped subscriber overlap, exception isolation, and the `clear()` teardown contract.
- [file] `reasons/network.py` — The TMS core where truth values live. Reading how `before`/`after` dicts are captured makes the `compute_changes` diff logic concrete.
- [general] `reasons_lib-vs-reasons-duplication` — The repo contains parallel `reasons/` and `reasons_lib/` trees with mirrored files. Understanding why both exist (vendoring? import-path shim? gradual migration?) prevents accidental edits to the wrong copy.
- [file] `reasons/mcp_server.py` — A likely consumer of the pub/sub events (pushing belief changes to MCP clients). Shows the "why" behind the API shape.

---

## Beliefs

- `pubsub-delivers-batched-final-state` — `publish()` is only ever called with the post-save stable snapshot; intermediate cascade states during fixpoint computation are structurally invisible to subscribers.
- `subscriber-exception-is-isolated` — A raising subscriber does not prevent other subscribers from receiving the same event batch, nor does it propagate to the caller of `publish()`.
- `subscribe-allows-duplicates` — Calling `subscribe(callback, db_path)` twice with the same callback registers it twice; `publish` will invoke it twice per event batch.
- `compute-changes-detects-insertions-as-changes` — A node present in `after` but absent from `before` produces a `ChangeEvent` with `old_truth_value=None`, so "new node" and "value changed" are indistinguishable by `type` alone (both are `"node_changed"`).
- `pubsub-is-synchronous-in-process` — There is no queue, no thread, no IPC; `publish()` blocks until all subscribers return, making it safe to call from any single thread but unsafe to call concurrently from multiple threads without external synchronization.
