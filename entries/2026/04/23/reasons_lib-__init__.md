# File: reasons_lib/__init__.py

**Date:** 2026-04-23
**Time:** 16:30

# `reasons_lib/__init__.py` — Data Model

## Purpose

This file defines the core data model for a **Reason Maintenance System** (RMS) based on Jon Doyle's 1979 Truth Maintenance System. It owns three dataclasses — `Justification`, `Node`, and `Nogood` — that represent the entire belief network structure. Every other module in `reasons_lib/` imports from here; this file imports nothing from the project itself. It is the foundational vocabulary that the rest of the system speaks.

The module acts as a pure data layer with no behavior — no validation, no state mutation, no I/O. It defines *what* a belief network looks like, while modules like `storage.py`, `network.py`, and `derive.py` handle persistence, graph operations, and truth propagation respectively.

## Key Components

### `Justification`
A single reason for a node to be believed. Two types exist:

- **SL (Support List)**: The standard justification. A node is supported iff every node in `antecedents` (the "inlist") is IN *and* every node in `outlist` is OUT. The outlist is what makes the system **non-monotonic** — it lets you express "believe X unless Y becomes believed."
- **CP (Conditional Proof)**: For consistency-based reasoning (belief is valid if assumptions don't lead to contradiction).

The `label` field provides a human-readable description of the justification's rationale.

### `Node`
A node in the dependency network representing a single belief/claim. Key fields:

| Field | Role |
|-------|------|
| `id` | Unique identifier (kebab-case string) |
| `text` | Human-readable description of the claim |
| `truth_value` | `"IN"` or `"OUT"` — the node's current status |
| `justifications` | List of `Justification` objects — a node is IN if **any** justification is valid |
| `dependents` | Reverse index — which other nodes reference this one in their justifications |
| `source` | Provenance (e.g., which file or agent produced this belief) |
| `source_hash` | Hash of the source content for staleness detection |
| `date` | When the node was created |
| `metadata` | Open-ended dict for extension without schema changes |

A **premise** is a node with an empty `justifications` list — it defaults to IN with no conditions.

### `Nogood`
A recorded contradiction: a set of node IDs that cannot all be IN simultaneously. The `resolution` field tracks how (or whether) the contradiction was resolved.

## Patterns

- **Pure data model at package root**: The `__init__.py` exports only dataclasses with no methods, making the data model importable as `from reasons_lib import Node, Justification, Nogood`. Behavior lives in sibling modules.
- **Default-IN convention**: `Node.truth_value` defaults to `"IN"`, which means newly created nodes (especially premises) are believed unless something retracts them. This matches Doyle's design where the system starts optimistic and retracts on contradiction.
- **Reverse index via `dependents`**: The `dependents` set on `Node` is a denormalized reverse pointer — when node A appears in node B's justification, B's id goes into A's `dependents`. This avoids full-graph scans during truth propagation.

## Dependencies

**Imports**: Only `dataclasses` from the standard library. Zero external or internal dependencies — this is intentionally at the bottom of the import graph.

**Imported by**: Every test module and (transitively) every module in `reasons_lib/`. This is the single shared vocabulary across the entire system. A change here ripples everywhere.

## Flow

There is no control flow in this file — it defines data structures only. The lifecycle of these objects is:

1. Constructed by `storage.py` (loading from SQLite) or by callers building networks programmatically
2. Manipulated by `network.py` and `derive.py` (truth propagation, dependency updates)
3. Serialized by `export_markdown.py` and `network.py` (for `beliefs.md` and `network.json`)
4. Checked by `check_stale.py` (comparing `source_hash` against current file contents)

## Invariants

- `Justification.type` is always `"SL"` or `"CP"` — but this is not enforced by the dataclass; it's a convention maintained by callers.
- `Node.truth_value` is always `"IN"` or `"OUT"` — again, enforced by the propagation logic in other modules, not here.
- A node with no justifications is a **premise** and is IN by default. The system treats the empty-justifications case specially.
- A node is IN if **any** justification is valid (disjunctive semantics). A justification is valid if **all** antecedents are IN and **all** outlist nodes are OUT (conjunctive semantics).
- `dependents` must be kept in sync with the justifications that reference a node. This is a manual invariant — nothing in this file enforces it.

## Error Handling

None. This file defines plain dataclasses with no validation, no `__post_init__`, and no type enforcement beyond Python's type hints. All validation and error handling happens downstream in the modules that consume these structures.

---

## Topics to Explore

- [file] `reasons_lib/derive.py` — Implements truth propagation: how IN/OUT status is recomputed when justifications change, including retraction cascades
- [file] `reasons_lib/storage.py` — How these dataclasses are serialized to/from SQLite (the `reasons.db` file), and whether schema constraints enforce any of the invariants this file doesn't
- [file] `reasons_lib/network.py` — Graph operations on the dependency network, including how `dependents` is maintained and how `network.json` is produced
- [function] `reasons_lib/check_stale.py:check_stale` — How `source_hash` on Node is used to detect beliefs that may be outdated relative to their source files
- [general] `doyle-1979-tms` — Doyle's original Truth Maintenance System paper; understanding SL/CP justification semantics and the non-monotonic reasoning model this code implements

## Beliefs

- `node-in-if-any-justification-valid` — A Node is IN when at least one of its Justifications is valid; validity requires all antecedents IN and all outlist nodes OUT
- `premises-have-no-justifications` — A premise node is represented by an empty `justifications` list and defaults to `truth_value="IN"`
- `init-is-pure-data-model` — `reasons_lib/__init__.py` contains only dataclass definitions with no behavior, validation, or I/O; it depends on nothing in the project
- `outlist-enables-non-monotonic-reasoning` — The `outlist` field on Justification allows beliefs to be retracted when a defeating node becomes IN, which is the core non-monotonic mechanism
- `dependents-is-manual-reverse-index` — The `Node.dependents` set is a denormalized reverse pointer that must be kept in sync by external code; nothing in this file enforces consistency

