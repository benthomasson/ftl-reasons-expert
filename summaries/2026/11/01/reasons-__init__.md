---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/__init__.py

# `reasons/__init__.py` — Core Data Model

## Purpose

This file is the **single source of truth for the domain data model** of the entire `reasons` package. It defines the three fundamental data structures that every other module in the system operates on: `Justification`, `Node`, and `Nogood`. There is no business logic here — only shape and semantics. Every module from `pg.py` (persistence) to `api.py` (HTTP endpoints) to `mcp_server.py` (MCP protocol) imports from this file to construct, deserialize, or validate these objects.

Because it lives in `__init__.py`, it's the first thing loaded when anyone does `from reasons import Node`. It's the contract that unifies the codebase.

## Key Components

### `Justification`

A reason *why* a node should be IN. Two semantic types:

| Field | Type | Meaning |
|-------|------|---------|
| `type` | `"SL"` or `"CP"` | SL = Support List (monotonic AND-gate); CP = Conditional Proof (assumption-based) |
| `antecedents` | `list[str]` | The **inlist** — node IDs that must all be IN |
| `outlist` | `list[str]` | Node IDs that must all be OUT (non-monotonic defeaters) |
| `label` | `str` | Human-readable description |
| `content_hash` | `str` | Content-addressed identifier (likely used for dedup) |

**SL semantics:** Node is IN iff `∀ a ∈ antecedents: a is IN` AND `∀ o ∈ outlist: o is OUT`.
**CP semantics:** Node is IN iff its assumptions are mutually consistent (checked elsewhere, not in this dataclass).

The `outlist` is what gives the system **non-monotonic reasoning** — "believe X *unless* Y is also believed." This is the Doyle (1979) defeat mechanism.

### `Node`

A belief in the network:

- `id` / `text` — identity and content.
- `truth_value` — `"IN"` or `"OUT"`. A **premise** (no justifications) is IN by default.
- `justifications` — list of `Justification`. Node is IN if **any** justification is valid.
- `supporting_justification` — index into `justifications` that currently holds. This is a cache to avoid re-evaluation.
- `dependents` — **reverse index** (set of node IDs that depend on this node). Critical for propagation: when a node flips OUT, you walk `dependents` to find what to invalidate.
- Source provenance fields: `source`, `source_url`, `source_hash`, `text_hash`.
- Lifecycle timestamps: `created_at`, `updated_at`, `reviewed_at`, `verified_at`, `retracted_at`.
- `metadata` — freeform dict for extension.

### `Nogood`

A recorded contradiction: a set of node IDs that **cannot all be IN simultaneously**.

- `id` — unique identifier.
- `nodes` — the conflicting set.
- `discovered` / `resolution` — audit trail.

Nogoods are the system's memory of "we already tried this combination; it fails." They prevent re-deriving contradictions.

## Patterns

1. **Flat dataclass model (no inheritance, no enums).** `type` is a plain `str` with documented allowed values. This keeps serialization (JSON, PG rows) trivial — no custom `__eq__`, no `Enum` import overhead. The tradeoff is that type safety is lost; callers must respect the `"SL"`/`"CP"` convention.

2. **Mutable-by-default dataclasses with `field(default_factory=...)`.** Every collection field uses `default_factory=list` or `default_factory=set`, which is the correct idiom for mutable defaults in dataclasses. The classes are *not* frozen — the system mutates `truth_value`, `justifications`, and `dependents` in-place during propagation.

3. **Reverse adjacency index (`dependents`).** Rather than scanning the entire graph on every flip, each node carries a `set` of node IDs that have it in their `antecedents` or `outlist`. This is the workhorse of the `derive.py` propagation loop.

4. **`reasons/` vs `reasons_lib/` duplication.** The repo has two parallel package trees. `reasons/__init__.py` is the "live" version; `reasons_lib/` appears to be a frozen/vendor copy (note that `reasons_lib` is missing `merkle.py`, `pubsub.py`, and `contradictions.py` — it's a subset). All the heavy importers target `reasons_lib`, suggesting the library is the stable API surface while `reasons/` is the evolving workspace.

5. **Content-addressing via hashes.** `content_hash`, `text_hash`, `source_hash` fields suggest nodes and justifications are deduplicated by hash rather than by ID alone.

## Dependencies

**Imports (outbound):**
- `dataclasses.dataclass`, `dataclasses.field` — the only external dependency. No third-party packages. This is intentional: the data model must remain importable in minimal environments (tests, CLI, MCP server) without pulling in `psycopg`, `httpx`, etc.

**Imported by (inbound):**
- Essentially **every** module in both `reasons/` and `reasons_lib/` (35+ modules), plus `tests/conftest.py` and 50+ test files.
- This makes it the **root of the dependency DAG**. A change to a field name or type here is a breaking change for the entire project.

## Flow

This file has **no execution flow** — it's pure data definition. The "flow" lives in consumers:

```
pg.py / storage.py   →  deserialize rows → Node / Justification objects
derive.py            →  read Node.justifications, evaluate inlist/outlist, flip truth_value, walk dependents
contradictions.py    →  detect Nogood violations, create Nogood records
mcp_server.py        →  serialize Node to JSON-RPC response
mcp_client.py        →  deserialize JSON-RPC payload → Node
```

The data model is the **wiring** between these: producers write fields, consumers read them. The `supporting_justification` index and `dependents` set are the hot-path optimizations that avoid O(n) scans during propagation.

## Invariants

These are rules the code *expects* but doesn't enforce via validation (dataclasses don't have `__post_init__` here):

1. **A node with zero `justifications` is a premise and is always IN.** (No code sets `truth_value = "OUT"` for a premise.)
2. **`supporting_justification`, if not `None`, must be a valid index into `justifications`.** Stale indices after list mutation are a class of bugs (see `tests/test_dangling_dependents.py`).
3. **`dependents` is a reverse index maintained by the mutator (likely `pg.py` or `derive.py`).** If you add a justification referencing node X, X's `dependents` set must gain the dependent's ID. Failure to do so breaks propagation.
4. **`truth_value` is constrained to `"IN"` or `"OUT"`.** No third state. (No tri-valued logic here despite Doyle's original allowing it.)
5. **`Nogood.nodes` is a set that cannot all be IN simultaneously.** The system must respect this during derivation — if a Nogood is about to be satisfied, the derivation must backtrack.
6. **`type` is `"SL"` or `"CP"`.** CP justifications are evaluated differently (assumption consistency) than SL (boolean inlist/outlist check).

## Error Handling

This file **produces no errors and handles none**. It's a passive data definition. All error handling lives downstream:

- **Deserialization errors** (bad JSON → `Node`): handled in `pg.py`, `storage.py`, `mcp_server.py`.
- **Invariant violations** (stale `supporting_justification`, missing dependents): detected in `derive.py`, `verify.py`, `repair.py`.
- **Contradiction detection** (Nogood satisfied): handled in `contradictions.py` and `derive.py`.

The absence of `__post_init__` validation is a deliberate choice: it keeps construction fast in the hot propagation loop where millions of nodes may be instantiated. Validation is pushed to the boundaries (API layer, persistence layer).

---

## Topics to Explore

- [file] `reasons/derive.py` — Implements the actual propagation algorithm that reads `Justification.inlist/outlist` and walks `Node.dependents`; understanding this shows how the data model drives runtime behavior.
- [file] `reasons/contradictions.py` — Shows how `Nogood` records are created and consulted; the "backtracking" mechanism that makes the system sound.
- [file] `reasons/pg.py` — The serialization/deserialization layer; reveals how these dataclasses map to PostgreSQL rows and where the `reasons` vs `reasons_lib` fork matters in practice.
- [general] `doyle-tms-semantics` — Doyle's 1979 Truth Maintenance System paper; the `outlist`/defeat mechanism and CP-vs-SL distinction here are direct implementations of his node justification types.
- [diff] `reasons/` vs `reasons_lib/` — The two parallel package trees differ in file coverage (e.g., `merkle.py`, `pubsub.py` exist only in `reasons/`); understanding why both exist and which is canonical is critical before making changes.

## Beliefs

- `reasons-__init__-is-single-source-of-truth-for-data-model` — All 35+ runtime modules and 50+ test files import `Node`, `Justification`, and `Nogood` exclusively from this module; no other file defines or re-exports these classes.
- `node-truth-value-is-binary` — `Node.truth_value` is constrained to `"IN"` or `"OUT"` with no third state; the system does not implement Doyle's original tri-valued (IN/OUT/undecided) logic.
- `dependents-set-enables-O(1)-invalidation` — `Node.dependents` is a reverse adjacency index maintained by the mutator; the propagation loop in `derive.py` uses it to find affected nodes without a full graph scan.
- `outlist-implements-non-monotonic-defeat` — A `Justification` with a non-empty `outlist` becomes invalid (and its node may flip OUT) when any outlist node transitions to IN; this is the system's defeasibility mechanism.
- `reasons-lib-is-frozen-subset-of-reasons` — `reasons_lib/` contains a strict subset of `reasons/` (missing `merkle.py`, `pubsub.py`, `contradictions.py`), suggesting it is a stable API snapshot while `reasons/` is the active development tree.
