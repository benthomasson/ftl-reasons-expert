---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/__init__.py

## `reasons/__init__.py` — Core Domain Model

### 1. Purpose

This file is the **shared data schema for the entire project**. It defines the three frozen-in-spirit dataclasses that represent the whole domain:

| Class | Role |
|---|---|
| `Justification` | One reason for believing a node (an inlist/outlist pair, or a conditional proof). |
| `Node` | A belief in the dependency network (premise or derived), with truth value, provenance, and lifecycle timestamps. |
| `Nogood` | A recorded contradiction — a set of node IDs that may not all be `IN`. |

It is deliberately **logic-free**. There are no methods, no `__post_init__`, no validation, and no serialization. Every behavior — propagation, hashing, storage, propagation, conflict detection — lives in sibling modules (`verify.py`, `merkle.py`, `storage.py`, `network.py`, `pg.py`, …). This module exists so those modules agree on one representation.

The docstring anchors the design: *"Reason Maintenance System — data model based on Doyle (1979)."* That's the Truth Maintenance System literature — nodes hold justifications, truth values propagate through dependency edges, and non-monotonic reasoning is expressed by "these must be OUT" lists.

### 2. Key Components

#### `Justification` (lines ~6–21)

```python
type: str                      # "SL" or "CP"
antecedents: list[str]         # inlist: IDs that must be IN
outlist: list[str]             # IDs that must be OUT
label: str
content_hash: str
```

Contract, as documented in the class docstring:

- **`SL` (Support List)** — the justified node is `IN` **iff every** node in `antecedents` is `IN` **and every** node in `outlist` is `OUT`.
- **`CP` (Conditional Proof)** — the node is `IN` **iff the assumptions are consistent**. `antecedents`/`outlist` are still carried but the validity test is external (see the handling in `verify.py` / `network.py`).

`outlist` is the TMS *defeater* list: "believe X unless Y is believed." When a node in `outlist` becomes `IN` the justification silently stops supporting the node, which is how the system retracts derivations without delete operations.

`content_hash` is carried on the justification itself, not just the node — meaning two justifications with identical antecedents/outlist/label but different hashes are considered distinct. It links to `reasons/merkle.py`.

#### `Node` (lines ~25–45)

```python
id: str
text: str
truth_value: str = "IN"                    # IN | OUT
justifications: list[Justification] = []
supporting_justification: int | None = None
dependents: set[str] = set()
source / source_url / source_hash / text_hash / date
metadata: dict = {}
created_at / updated_at / reviewed_at / verified_at / retracted_at: str
```

Notable fields:

- **`truth_value` defaults to `"IN"`.** This reflects the docstring rule *"A premise has no justifications and is IN by default."* A node with justifications must have its truth value recomputed by propagation code — the default is only correct for premises.
- **`supporting_justification` is an *index*, not an ID.** `# index into justifications list`. This is the single most dangerous field in the file: it depends on list order. Any code that inserts/removes/reorders `justifications` must recompute or invalidate this index (`test_add_justification.py`, `test_remove_justification.py` exist precisely for this).
- **`dependents: set[str]`** is a *denormalized reverse index* — every node ID whose `antecedents` or `outlist` mentions this node's `id`. It is never populated or validated in this file; callers own it.
- **Timestamps are plain `str`**, presumably ISO-8601, with no format enforcement. `retracted_at` being non-empty is the retraction signal (the field doubles as a boolean).
- **`source*` / `text_hash` / `date`** carry provenance and deduplication keys — see `import_beliefs.py`, `test_deduplication.py`, `test_pin_sources.py`.
- **`metadata: dict`** is untyped on purpose; it is the extension point for access tags, topics, LLM metadata, etc. (`reasons/metadata.py`, `test_access_tags.py`).

#### `Nogood` (lines ~47–52)

```python
id: str
nodes: list[str] = []      # IDs that cannot all be IN simultaneously
discovered: str = ""
resolution: str = ""
```

A *nogood* in the CSP/TMS sense: an at-most-one-false constraint over a set of nodes. `resolution` records the free-form description of how the conflict was resolved (or empty if unresolved). The `nodes` list order is not semantically meaningful — it's a set encoded as a list for JSON friendliness.

### 3. Patterns

1. **Passive data object / anemic domain model.** All three classes are pure data; behavior is externalized. This makes storage backends, JSON round-trip, and Postgres mapping trivial (`reasons/pg.py`, `reasons/storage.py` simply map fields to columns).
2. **`field(default_factory=...)` for every mutable default** — `list`, `set`, `dict`. Standard Python idiom to avoid shared mutable defaults; also means two freshly-constructed `Node`s never alias one another's `justifications` or `dependents`.
3. **Stringly-typed enumerations.** `type ∈ {"SL","CP"}` and `truth_value ∈ {"IN","OUT"}` are plain `str`, not `Enum`s. This is a deliberate convenience for JSON/Postgres/`str` comparison — it permits any caller to write `node.truth_value == "IN"` without imports, at the cost of typos being caught only at runtime.
4. **Default-logic encoding of non-monotonicity** (Doyle/Poole style) — the inlist/outlist pair is the structural representation of "conclude X unless Y."
5. **Denormalized reverse index** (`dependents`) — trades consistency burden for O(1) "who depends on me?" queries during propagation.
6. **Index-instead-of-reference** (`supporting_justification`) — avoids cycle problems in serialization (no reference to a `Justification` object) but couples to list ordering.

### 4. Dependencies

**Outbound:** exactly one — `from dataclasses import dataclass, field` (stdlib). Zero third-party, zero intra-repo. That is itself the point: this file can be imported from anywhere without risk of a circular import.

**Inbound:** essentially the whole repository. Both source trees import it:

- `reasons/mcp_server.py`, `reasons/pg.py`
- every module in `reasons_lib/*` (api, ask, build_wiki, check_stale, cli, cluster, compact, contradictions, derive, export_card, export_markdown, hf, import_agent, import_beliefs, llm, mcp_client, mcp_server, metadata, mind_service, network, pg, propose_update, pubsub, repair, repair_premises, report_belief, report_gated, review, review_justifications, review_premises, storage, verify)
- `tests/conftest.py` plus ~all 60 test files

Because both `reasons/` and `reasons_lib/` contain the *same module names*, this file is almost certainly the **canonical schema** that `reasons_lib` imports from rather than redefining. That is why `reasons_lib/__init__.py` appears in the imported-by list.

### 5. Flow

There is **no runtime control flow**. Importing the module executes only:

1. The module docstring and `dataclasses` import.
2. Three `@dataclass` decorators, which generate `__init__`, `__repr__`, `__eq__` per class.

All execution is deferred to callers. The typical lifecycle a developer will encounter:

```
Nogood / Node / Justification constructed in storage.py or import_beliefs.py
        ↓
truth values propagated in verify.py / network.py
        ↓
supporting_justification set to the winning index
        ↓
dependents sets maintained as edges are added/removed
        ↓
serialized by pg.py / storage.py / export_markdown.py
```

### 6. Invariants

None of these are enforced in this file — they are conventions every consumer relies on:

| Invariant | Where it must be enforced |
|---|---|
| `Justification.type ∈ {"SL","CP"}` | construction sites (`add_justification` path) |
| `Node.truth_value ∈ {"IN","OUT"}` | `verify.py` propagation |
| If `supporting_justification is not None`, it is a valid, in-range index into `justifications` | add/remove-justification mutation code |
| `justifications` list ordering is stable whenever `supporting_justification` refers to it | same |
| A premise = `justifications == []` and `truth_value == "IN"` | construction sites |
| Node `id`s are unique within a store | `storage.py` / `pg.py` |
| `Node.dependents` mirrors the inverse of all `antecedents`+`outlist` lists | mutation code; guarded by `tests/test_dependents_integrity.py` |
| Every `id` in `Justification.antecedents`, `outlist`, `Nogood.nodes`, `Node.dependents` refers to an existing node | `storage.py`; `tests/test_dangling_dependents.py` |
| `Nogood.nodes` contains distinct IDs | construction sites; `tests/test_nogood_id.py` |
| Timestamps are ISO-8601 strings; empty string means "never" | callers |

The `Node.truth_value` default of `"IN"` and the empty-`justifications` premise rule interact: a `Node()` with justifications but left at the default will be silently wrong until propagation runs.

### 7. Error Handling

**There is none.** This is intentional for a pure schema module:

- No `__post_init__` hooks, so no construction-time validation — you can build `Justification(type="banana")`, `Node(truth_value="MAYBE")`, or a `Node` whose `supporting_justification=99` with two justifications, and nothing raises.
- No `TypeError` guards for wrong types beyond what the (runtime-irrelevant) annotations imply.
- Failures surface **downstream** and typically as `IndexError` (bad `supporting_justification`), `KeyError` (dangling antecedent ID during propagation), or silent incorrect results (unknown `truth_value` falls through comparison branches that test `== "IN"`).

A maintainer adding validation here should be aware that every module in the repo constructs these dataclasses directly, so tightening `__post_init__` would be a repo-wide breaking change.

---

## Topics to Explore

- [file] `reasons/verify.py` — Implements the propagation rules this file merely declares: how `truth_value` is recomputed from `justifications`, how `SL` vs `CP` are evaluated, and how `supporting_justification` is selected.
- [file] `reasons_lib/__init__.py` — Determines whether `reasons_lib` re-exports these classes or wraps them, explaining the duplicated module tree and the import direction between `reasons/` and `reasons_lib/`.
- [file] `tests/test_dependents_integrity.py` — Documents what "consistent reverse index" means operationally and which mutations can break `dependents`.
- [function] `reasons/storage.py` — Look for the load/save path to see how `set[str]` (`dependents`) and nested `list[Justification]` are serialized and reconstructed, and whether `supporting_justification` indices survive a round-trip.
- [general] `supporting-justification-index-lifecycle` — How `supporting_justification` is kept valid across add/remove operations (see `tests/test_add_justification.py` and `tests/test_remove_justification.py`); this is the most likely source of latent bugs in the model.

## Beliefs

- `justification-type-is-a-string-discriminator` — `Justification.type` is a plain `str` restricted by convention to `"SL"` or `"CP"`; the module defines no Enum and performs no validation.
- `supporting-justification-contains-list-index` — `Node.supporting_justification` is an integer index into `Node.justifications`, not an ID or object reference, so the ordering of the `justifications` list is semantically meaningful.
- `dependents-is-a-denormalized-reverse-index` — `Node.dependents` is a `set[str]` that must mirror the inverse of all `antecedents` and `outlist` edges; this module never populates or checks it.
- `module-is-a-validation-free-schema-layer` — The file defines no `__post_init__`, no methods, and no validation, so invalid states such as an out-of-range `supporting_justification` or an unknown `truth_value` are constructible without error.
- `outlist-enables-non-monotonic-retraction` — `Justification.outlist` encodes defeaters: a justification is invalidated whenever any node in `outlist` becomes `IN`, which is what allows beliefs to be retracted without deletion.
- `node-defaults-to-in-for-premises` — `Node.truth_value` defaults to `"IN"` and a premise is defined as a `Node` with an empty `justifications` list, so freshly constructed premises are `IN` without any propagation pass.
