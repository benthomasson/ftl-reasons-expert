---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Understand the TMS cascade model and why only final stable states are published.

# TMS Cascade Model and Final State Publication

## The Cascade Model

The TMS (Truth Maintenance System) is a belief network where every node has a truth value of `IN` (believed) or `OUT` (retracted). Three structural elements drive propagation:

**1. Node types** (`README.md`)
- **Premises** (depth-0): observable facts, no justifications
- **Derived** (depth-1+): dependent on other nodes via justifications
- **Outlist-gated**: "believe X *unless* Y is IN" — non-monotonic defaults

**2. Justification semantics** (`README.md`)
- `SL` (Support List): node is `IN` when **all** antecedents are `IN`
- A node with **any** valid justification is `IN` (multiple justifications act as disjuncts)
- Outlist: justification valid only when all inlist nodes are `IN` *and* all outlist nodes are `OUT`

**3. The cascade** (`README.md`, `reasons/derive.py:14-38`)
- When a premise goes `OUT`, all dependents whose justifications become invalid cascade `OUT`
- When a premise returns `IN`, dependents are automatically **restored — without rederivation**
- Nogoods (contradiction sets) trigger **dependency-directed backtracking** to retract the minimal responsible premise

The API exposes cascade effects for preview and execution — `retract` and `assert` (`reasons/api.py:475,757`) both compute cascades, and `what_if_retract` / `what_if_assert` (`reasons/api.py:517,585`) return the cascade without mutating the database. Each returned cascade is split into `went_out` and `went_in` sets (`reasons/cli.py:114-115`) so callers can see both directions of propagation.

## Why Only Final Stable States Are Published

The eventing layer is explicitly designed around the *committed* network state, not intermediate propagation steps. Three observations explain why:

**1. The pubsub contract is explicit about it.**
`reasons/pubsub.py:4-5` states it directly:

> *"Subscribers receive batched events after each `_with_network` write transaction completes (after save). Events are never emitted for intermediate cascade states — only the final stable state is visible."*

Same module, line 71-72: `publish()` is *"Called by `_with_network.__exit__` after save completes."*

**2. Persistence is a single atomic rewrite.**
`reasons/storage.py:3` — *"ACID transactions ensure propagation cascades are atomic."* The `save()` method (`reasons/storage.py:190+`) wraps the entire network rewrite in `with self.conn:` (a single SQLite transaction), which `DELETE`s every table and re-`INSERT`s the final in-memory `Network`. Intermediate propagator states never touch disk.

**3. Events are derived from before/after *snapshots*, not from the propagator.**
`reasons/pubsub.py:96-119` — `compute_changes(before, after, db_path)` diffs two full truth-value snapshots. The `ChangeEvent` dataclass (`reasons/pubsub.py:20-31`) explicitly *lacks* `cause` and `cascade_root` fields because the hook only sees the pre/post snapshots — it does not observe the propagator's intermediate steps. `reasons/api.py:158` is the integration point: `pubsub.publish(events, db_path)` fires once per write.

### The mechanism, concretely

```
_with_network (context manager)
   ├─ __enter__:  snapshot_before = {node_id: truth_value, ...}
   ├─ body:       cascades mutate network.nodes in memory
   │              (many transient IN→OUT→IN states possible here)
   ├─ save():     atomic SQLite transaction commits FINAL state
   └─ __exit__:   after = snapshot_before-after save
                  events = compute_changes(before, after, db_path)
                  pubsub.publish(events, db_path)
```

Because `compute_changes` only reads the *snapshot* before entry and the *snapshot* after save, any node that flips `IN → OUT → IN` during propagation produces **zero** events — the diff sees `IN == IN`. A node that ends in a different state from where it started produces exactly one event. This is consistent with the semantic argument: subscribers care about *what the network now believes*, not the path the propagator took to get there. It also matches the atomicity guarantee — a subscriber that observed a half-propagated state could make decisions on an inconsistent snapshot that never existed in the database.

## What the Observations Don't Cover

To confirm the picture fully, we'd want to read:

- The actual `_with_network` implementation (not shown — likely in `reasons/api.py`, referenced but not quoted) to verify snapshot timing and whether `before` is captured pre- or post-propagation.
- The propagation algorithm itself (`Network.propagate` or equivalent) — the observations don't show how the fixpoint is reached, only that restoration happens automatically.
- Whether the proposal accept/reject flow (`reasons/api.py:4471-4531`, `reasons/cli.py:2816`) uses the same `_with_network` path or a separate query-only path (the `what_if_*` functions look read-only and shouldn't publish).
- Any batching/coalescing logic between rapid successive `_with_network` calls (not evident from `pubsub.py`).

## Topics to Explore

- [function] `reasons/api.py:_with_network` — The context manager that captures snapshots, runs propagation, saves atomically, and drives `pubsub.publish`; critical for confirming the before/after diff semantics.
- [file] `reasons/network.py` — Where the propagation fixpoint and SL/outlist justification evaluation actually live; explains how cascades reach stable states.
- [function] `reasons/pubsub.py:compute_changes` — The single source of "what changed" semantics; explains why `IN → OUT → IN` yields no event.
- [function] `reasons/api.py:what_if_retract` — The read-only cascade preview; contrast with mutating `retract` to see what *would* be published before committing.
- [file] `reasons/storage.py` — The `save()` method that wraps network persistence in a single SQLite ACID transaction, enforcing atomicity of cascades.

## Beliefs

- `pubsub-publishes-only-final-stable-state` — `reasons/pubsub.py` documents that subscribers receive events only after `_with_network` commits, never for intermediate cascade states.
- `network-persistence-is-atomic` — `Storage.save()` in `reasons/storage.py` wraps the full network rewrite in a single SQLite `with self.conn:` transaction.
- `change-events-are-snapshot-diffs` — `compute_changes` in `reasons/pubsub.py:96` derives events by diffing truth-value snapshots rather than observing the propagator.
- `cascades-produce-both-out-and-in-sets` — `reasons/api.py:517-627` and `reasons/cli.py:114-115` split cascade results into `went_out` and `went_in`.
- `cascade-depth-is-computed-per-target` — `_cascade_depth` (`reasons/api.py:737`) measures how far a target node sits from the retracted node, used to annotate cascade effects.
