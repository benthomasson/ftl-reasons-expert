---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Explore how source, `beliefs_type`, and dependent count combine into `_entrenchment`.

## How `_entrenchment` combines source, `beliefs_type`, and dependent count

`_entrenchment` is an integer ranking score for a node/premise. It exists in two parallel implementations:

- In-memory network: `reasons/network.py:291` — `def _entrenchment(self, node_id: str) -> int:`
- PostgreSQL-backed store: `reasons/pg.py:3328` — `def _entrenchment(self, cur, node_id):`

Both are called while computing retraction/resolution candidates: `reasons/network.py:368` and `reasons/pg.py:3316`.
The surrounding network path returns both `dependent_count` and `entrenchment` per premise (`reasons/network.py:345`, `reasons/network.py:372`).

### 1. Source

`source` is a first-class node field in storage: the SQLite schema in `reasons/storage.py` defines `nodes.source`, `source_url`, and `source_hash`.

The observed test suite shows source presence raises entrenchment:

- `tests/test_backtracking.py:259` — `assert net._entrenchment("sourced") > net._entrenchment("unsourced")`
- `tests/test_backtracking.py:230` — `assert net._entrenchment("evidence") > net._entrenchment("basis")`

So a sourced/evidence-bearing node gets a higher score than an unsourced/basis node. The exact field checked and the size of the bonus are not visible in the observations.

### 2. `beliefs_type`

`beliefs_type` is stored in per-node metadata and is normalized to uppercase before use:

- `reasons/network.py:321` — `btype = node.metadata.get("beliefs_type", "").upper()`
- `reasons/pg.py:3361` — `btype = meta.get("beliefs_type", "").upper()`

Missing `beliefs_type` becomes `""`. Importers populate it from the imported claim’s `type` field:

- `reasons/import_agent.py:108`
- `reasons/import_beliefs.py:248`
- `reasons/pg.py:2462`

The test suite confirms that type affects the score:

- `tests/test_backtracking.py:266` — `assert net._entrenchment("obs") > net._entrenchment("der")`
- `tests/test_backtracking.py:264-265` create `OBSERVATION` and `DERIVED` nodes.

So `OBSERVATION` outranks `DERIVED` in entrenchment. Because the value is uppercased, the comparison is case-insensitive.

### 3. Dependent count

Dependent count is derived from graph topology, not passed in as a parameter.

In the in-memory network path:

- `reasons/network.py:368` calls `self._entrenchment(premise_id)`
- `reasons/network.py:372` computes `dependent_count` as `len(self.nodes[premise_id].dependents)`

In the PostgreSQL path:

- `reasons/pg.py:3316` calls `self._entrenchment(cur, premise_id)`
- `reasons/pg.py:3321` computes `"dependent_count": len(deps)`
- `reasons/pg.py:551` shows dependent discovery via `_find_dependents`

The test at `tests/test_backtracking.py:275` compares `popular` vs `lonely`, which is consistent with more dependents producing higher entrenchment: a premise many beliefs depend on is more entrenched than an isolated one.

### How the three combine

The observations establish the inputs and the ordinal behavior:

- source present > source absent
- `OBSERVATION` > `DERIVED`
- more dependents > fewer dependents

They do **not** show the exact arithmetic. The full `_entrenchment` bodies are missing, so we cannot say whether the combination is an additive weighted sum, a step function, capped, or something else. What is clear is that `_entrenchment` is a graph-sensitive integer score that mixes node provenance (`source`), node metadata (`beliefs_type`), and topology (`dependent_count`/`dependents`).

### What is missing

To state the exact formula, we would need:

- the full body of `reasons/network.py:_entrenchment` starting at line 291
- the full body of `reasons/pg.py:_entrenchment` starting at line 3328
- the setup for `popular`/`lonely` and `sourced`/`unsourced` in `tests/test_backtracking.py`
- whether PG uses the same weights and source semantics as the in-memory network

## Topics to Explore

- [file] `reasons/network.py` — Read the full `_entrenchment` body around line 291 to see the exact source, `beliefs_type`, and dependent-count weights.
- [function] `reasons/pg.py:_entrenchment` — Compare the SQL-backed implementation with the in-memory one and check for parity.
- [file] `tests/test_backtracking.py` — Read lines 230-275 to see how `sourced`, `unsourced`, `obs`, `der`, `popular`, and `lonely` are constructed.
- [general] `entrenchment-parity-network-vs-pg` — Verify whether the two `_entrenchment` implementations produce identical scores for the same node data.
- [function] `reasons/pg.py:_find_dependents` — Understand how PG computes the dependent set that feeds `len(deps)` and likely the entrenchment score.

## Beliefs

- `_entrenchment-is-implemented-in-network-and-pg` — Both `reasons/network.py:291` and `reasons/pg.py:3328` define an `_entrenchment` function returning an integer score.
- `beliefs-type-is-read-from-node-metadata` — `_entrenchment` reads `beliefs_type` from node metadata and uppercases it at `reasons/network.py:321` and `reasons/pg.py:3361`.
- `source-presence-increases-entrenchment` — `tests/test_backtracking.py:259` asserts sourced nodes have higher entrenchment than unsourced nodes.
- `observation-outranks-derived-in-entrenchment` — `tests/test_backtracking.py:266` asserts an `OBSERVATION` node has higher entrenchment than a `DERIVED` node.
- `dependent-count-is-len-dependents-in-network-path` — `reasons/network.py:372` computes dependent count as `len(self.nodes[premise_id].dependents)` in the same retraction path that calls `_entrenchment`.
