# File: reasons_lib/__init__.py

**Date:** 2026-05-05
**Time:** 15:30

# `reasons_lib/__init__.py` — Data Model

## Purpose

This is the domain model layer for the entire reason maintenance system. It defines the three core data classes — `Justification`, `Node`, and `Nogood` — that every other module in `reasons_lib` and virtually every test file depends on. It owns no logic, no I/O, and no state management; its sole responsibility is defining the shape of data that flows through the system.

The module docstring names the lineage explicitly: Doyle's 1979 Truth Maintenance System (TMS). That's not decorative — the field names (`inlist`, `outlist`, `antecedents`, `dependents`, truth values `IN`/`OUT`) are drawn directly from the TMS literature, and understanding the paper helps you understand why the model looks the way it does.

## Key Components

### `Justification`

A single reason why a node might be believed. Two types:

- **SL (Support List)**: The workhorse. A node is justified when all `antecedents` are IN *and* all `outlist` nodes are OUT. The outlist is what makes the system non-monotonic — it encodes "believe X unless Y."
- **CP (Conditional Proof)**: For consistency-based reasoning. Less commonly used in this codebase.

The `label` field is an optional human-readable tag. The `type` field is a bare string (`"SL"` or `"CP"`), not an enum — consumers must validate.

### `Node`

A proposition in the dependency network. Key fields:

| Field | Role |
|---|---|
| `id` | Unique kebab-case identifier, used as a key everywhere |
| `text` | Human-readable description of the claim |
| `truth_value` | `"IN"` or `"OUT"` — defaults to `"IN"` |
| `justifications` | List of `Justification` objects — if empty, the node is a **premise** (believed axiomatically) |
| `dependents` | Reverse index: IDs of nodes whose justifications reference this node |
| `source`, `source_url`, `source_hash`, `date` | Provenance tracking |
| `metadata` | Open-ended dict for extensions (access tags, etc.) |

The distinction between premises and derived nodes is implicit: a node with an empty `justifications` list is a premise. There's no explicit flag.

### `Nogood`

A recorded contradiction — a set of node IDs that cannot all be IN simultaneously. The `resolution` field tracks how (or whether) the conflict was resolved. Nogoods are the system's mechanism for detecting and recording logical inconsistencies.

## Patterns

**Pure data classes with no behavior.** This is an anemic domain model by design — all logic (propagation, retraction, staleness checking) lives in other modules (`network.py`, `storage.py`, `check_stale.py`). The data classes are deliberately kept as dumb containers so they can be serialized/deserialized freely (JSON, SQLite, Markdown).

**Mutable defaults via `field(default_factory=...)`**. Every collection field uses `default_factory` to avoid the shared-mutable-default trap. This is the standard Python dataclass idiom.

**Stringly-typed enums.** Both `Justification.type` (`"SL"`/`"CP"`) and `Node.truth_value` (`"IN"`/`"OUT"`) are plain strings. This simplifies serialization but pushes validation to consumers.

## Dependencies

**Imports:** Only `dataclasses` from the standard library. Zero external dependencies. This is the leaf of the dependency tree.

**Imported by:** Every test file and (transitively) every module in `reasons_lib`. This file is the most-imported module in the project — 29 test files reference it directly. Changes here propagate everywhere.

## Flow

There is no control flow. These are pure data definitions. Instances are created by:

1. **`storage.py`** — deserializes from SQLite rows
2. **`network.py`** — builds nodes programmatically and runs propagation
3. **`import_beliefs.py` / `import_agent.py`** — constructs nodes from external formats
4. **Test fixtures** (`conftest.py`) — creates nodes/justifications for test scenarios

## Invariants

The data classes themselves enforce almost nothing — the invariants are maintained by convention and by consuming code:

- A `Node` with no justifications is a premise and should be IN by default.
- A `Justification` of type `"SL"` is valid iff all `antecedents` are IN and all `outlist` nodes are OUT.
- A `Node` is IN iff at least one of its justifications is valid (disjunctive semantics).
- `dependents` must be kept in sync with justification references — if node A appears in node B's antecedents or outlist, then A's `dependents` set should contain B's id. This is not enforced here; `network.py` manages it.
- `Nogood.nodes` should contain at least two IDs (a single-node contradiction is meaningless), but this isn't validated.

## Error Handling

None. These are plain dataclasses. No validation, no exceptions, no type checking at construction time. Invalid states (e.g., `truth_value="MAYBE"`, `type="XYZ"`) are representable and must be caught upstream.

## Topics to Explore

- [file] `reasons_lib/network.py` — Implements the propagation algorithm that evaluates justifications and flips truth values; this is where the data model comes to life
- [file] `reasons_lib/storage.py` — Handles SQLite serialization/deserialization of these dataclasses; understanding the schema clarifies field semantics
- [function] `reasons_lib/check_stale.py:check_stale` — Detects when a node's truth value is inconsistent with its justifications, the main consumer of the validity invariants described above
- [general] `outlist-semantics` — The outlist mechanism is the key to non-monotonic reasoning; trace how outlist nodes in justifications trigger retraction cascades when they go IN
- [file] `tests/conftest.py` — Shows how the test suite constructs nodes and networks; good for understanding typical usage patterns of these data classes

## Beliefs

- `node-premise-is-implicit` — A node is a premise (believed without justification) iff its `justifications` list is empty; there is no explicit `is_premise` flag
- `node-truth-disjunctive` — A node is IN if ANY of its justifications is valid; this disjunctive semantics means adding a justification can never make a node go OUT
- `dependents-not-self-enforced` — The `Node.dependents` reverse index is not maintained by the data model itself; `network.py` is responsible for keeping it consistent with justification references
- `init-has-zero-external-deps` — `reasons_lib/__init__.py` depends only on `dataclasses` from the standard library, making it the dependency-free root of the package

