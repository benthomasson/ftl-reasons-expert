---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Whether `sl=` edges are validated (no self-loops, no cycles) at write time or only walked defensively at read time.

## Short answer

On the evidence provided, `sl=` antecedents are **parsed and persisted without an obvious write-time self-loop or cycle check in the API/storage layers**. The cycle-safety evidence is primarily **defensive graph-walk logic at read/traversal time**, e.g. `memo[...] = 0  # cycle guard` in traversal code.

The caveat is that the bodies of `Network.add_justification` and `_justification_valid` were not included, so a write-time rejection could still exist inside the network layer. But the API parsing layer and SQLite schema shown do not enforce “no self-loops, no cycles.”

## Write path

`sl=` is parsed as a comma-separated antecedent list in the API layer:

- `reasons/api.py:316-323` — `add_node` passes `sl=sl` into the network mutation, then does:
  - `antecedents = [a.strip() for a in sl.split(",")]`
- `reasons/api.py:435-440` — `add_justification` does the same parsing.

There is no visible `if node_id in antecedents: raise ...` check in those API blocks, and no visible cycle traversal before calling the network layer.

The persistence schema also does not enforce cycles or self-loops:

- `reasons/storage.py` defines `justifications.antecedents_json TEXT NOT NULL DEFAULT '[]'`.
- `justifications.node_id` has a foreign key to `nodes(id)`, but individual antecedents are stored inside JSON, so SQLite cannot enforce per-antecedent referential integrity, self-loops, or cycles.
- No trigger/check constraint for antecedent graph acyclicity is shown.

So a call like `add_node("a", "...", sl="a")` would, at least at the API parsing/storage level, produce a justification whose antecedent list contains the node itself. Whether `Network.add_node` / `Network.add_justification` rejects it is not shown.

## Read/walk path

There is evidence of defensive cycle handling during traversal:

- `reasons/api.py:3378` contains `memo[nid] = 0  # cycle guard`.
- `reasons/derive.py:_get_depth` also uses a `memo` cycle guard before walking antecedents.
- `tests/test_backtracking.py` exercises `trace_assumptions` and `find_culprits` over chains and diamonds, but the shown tests are all acyclic and do not assert self-loop/cycle rejection behavior.

That means the read/walk side is at least partially cycle-aware, but the write side is not clearly validated in the provided observations.

## Missing evidence

To be conclusive, we would need the bodies of:

- `reasons/network.py:add_justification`
- `reasons/network.py:_justification_valid`
- `reasons/pg.py:_justification_valid_cached`

Those could contain write-time validation. But based on the API parsing and SQLite schema observed, there is no explicit no-self-loop/no-cycle enforcement at the API or storage layer.

## Topics to Explore

- [function] `reasons/network.py:add_justification` — Check whether the network layer rejects self-referential antecedents or cyclic justifications before mutation.
- [function] `reasons/network.py:_justification_valid` — Determine whether “valid” means antecedent existence/truth-value validity only, or also graph acyclicity.
- [function] `reasons/pg.py:_justification_valid_cached` — Verify whether the PostgreSQL backend provides additional write-time validation not present in SQLite.
- [file] `reasons/storage.py` — Confirm there are no triggers or constraints enforcing acyclic antecedent graphs over `antecedents_json`.
- [file] `tests/test_backtracking.py` — See whether tests cover cyclic/self-loop justification graphs; the shown tests are acyclic.

## Beliefs

- `sl-parsing-in-api-add-node-splits-comma-separated-antecedents` — `api.add_node` parses `sl` into `antecedents = [a.strip() for a in sl.split(",")]` at `reasons/api.py:322-323`.
- `sl-parsing-in-api-add-justification-splits-comma-separated-antecedents` — `api.add_justification` parses `sl` the same way at `reasons/api.py:439-440`.
- `storage-schema-stores-antecedents-as-json-not-relational-edges` — `reasons/storage.py` stores antecedents in `justifications.antecedents_json TEXT`, so SQLite cannot enforce per-antecedent self-loops or cycles.
- `api-traversal-has-cycle-guard` — `reasons/api.py:3378` contains `memo[nid] = 0  # cycle guard`.
- `derive-depth-walk-has-cycle-guard` — `reasons/derive.py:_get_depth` uses a `memo` guard while walking antecedent depth.
