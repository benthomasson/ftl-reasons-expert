---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: The broader RMS semantics of justifications, antecedents, outlists, nogoods, and backtracking that this propagation implements.

# Broader RMS Semantics of This Propagation

The observations sketch a classic **Reason Maintenance System (RMS)** in the Doyle / de Kleer style: three element types (nodes, justifications, nogoods), a non-monotonic belief rule (SL / outlist / GATE), and dependency-directed backtracking when contradictions are detected. Below is a synthesized view of what the code actually implements, tied to the specific files and symbols we can see.

## 1. The three node kinds and the SL justification rule

The `derive.py` prompt (`reasons/derive.py`, `DERIVE_PROMPT`, ~lines 30–110) is the clearest documentation of the semantics:

> "A Reason Maintenance System (RMS) tracks beliefs with justifications and automatic retraction cascades. There are three kinds of nodes:
> 1. **Base premises** (depth-0): Observable facts with no justifications
> 2. **Derived conclusions** (depth-1+): Justified by antecedents via SL (support-list) rules
> 3. **Outlist-gated conclusions**: Justified by antecedents UNLESS certain nodes are IN"

A **justification** is an SL rule with two node lists (both stored in `justifications` per `reasons/storage.py`, schema at `SCHEMA`):

| Field | Semantics |
|---|---|
| `antecedents` (inlist) | All must be **IN** |
| `outlist` | All must be **OUT** |
| `type` | `"SL"` today (the API's `--unless` creates an SL justification too, per `tests/test_outlist.py::TestOutlistAPI::test_unless_only_no_sl`) |
| `label` | Human-readable rationale |
| `content_hash` | Dedup / change detection |

A node is **IN** iff *at least one* of its justifications is valid. `tests/test_outlist.py::test_alternate_justification_survives_outlist` exercises exactly this: X has two SL justifications, Y goes IN so the first fails, but the second (SL(A)) keeps X IN.

The **depth** of a node is computed in `reasons/derive.py::_get_depth` (lines ~120–150) — premises get depth 0, and each derived node is `1 + max(depth of antecedents in its active justification)`. This is the same traversal that `trace_assumptions` uses for backtracking.

## 2. Outlists = non-monotonic "unless" (GATE)

`tests/test_outlist.py` opens with the exact semantics:

> "Doyle's key innovation: SL justifications have both an inlist (must be IN) and an outlist (must be OUT). 'Believe X unless Y is believed.'"

Concrete properties encoded in tests:

- **Validity table** (`TestOutlistBasic`):
  - `Y` OUT, no antecedents → `X` is IN
  - `Y` IN, no antecedents → `X` is OUT
  - `A` IN, `Y` OUT → `X` is IN (both conditions hold)
  - `A` OUT, `Y` OUT → `X` is OUT (inlist fails)
- **Absent outlist nodes are treated as OUT** (`test_outlist_node_not_in_network`). The semantics of "not believed" toggle the default to IN — the system won't block a default just because the exception node hasn't been introduced yet.
- **Non-monotonic flipping** (`TestOutlistCascading`): asserting an outlist node retracts the dependent; retracting the outlist node restores it. The cascade walks back out through **all transitive dependents** (`test_outlist_cascade_through_chain` shows X→Z both flipping).
- **Multiple outlist nodes are conjunctive**: all must be OUT for the justification to hold (`test_multiple_outlist_nodes`).
- **Explain traces blame the outlist** rather than antecedents when the justification fails on outlist grounds: `net.explain("x")` returns `violated_outlist == ["y"]` (`TestOutlistExplain`). This is important — the RMS wants to distinguish "not justified because antecedent missing" from "suppressed by counter-evidence."
- **`--unless` at the API** (`TestOutlistAPI::test_add_with_unless`) is sugar for creating an SL justification with empty inlist and the given outlist. Because `type` is still set to `"SL"` (per `test_unless_only_no_sl`), outlist gating is *not* a separate justification type — it's a feature of SL.

The `derive.py` prompt maps outlist semantics onto the higher-level "GATE belief" concept: propose a positive conclusion with antecedents + `Unless: <negative claim>` (see `DERIVE_PROMPT` and `parse_proposals`, which parses both `### DERIVE` and `### GATE` blocks and fills a `"unless"` key alongside `"antecedents"`).

## 3. Nogoods = declared contradictions

A **nogood** is an explicit assertion that a set of nodes cannot all be IN. The data model is in `reasons/storage.py` (`nogoods` table: `id`, `nodes_json`, `discovered`, `resolution`).

**Content-derived IDs** (`tests/test_nogood_id.py`):
- `add_nogood(["b", "a"]) → "nogood-a-b"` (nodes sorted, dash-joined)
- Duplicates get numeric suffixes (`nogood-a-b-2`)
- Three-node nogoods use all three (`nogood-x-y-z`)
- The ID survives save/load, and `import_json` / `import_beliefs` preserve legacy-format IDs by *re-running* `add_nogood` on their node lists (`test_import_preserves_old_format_ids`)

**Activation condition**: A nogood only fires when *all* of its nodes are IN. If even one is already OUT, `add_nogood` is a no-op — `test_inactive_nogood_no_backtracking` returns `changed == []` and appends zero `backtrack` log entries.

## 4. Dependency-directed backtracking

The tests in `tests/test_backtracking.py` describe the full loop, and it is the most nuanced part of the RMS:

### 4a. `trace_assumptions(node_id) → list[premise_id]`
Walks **backward** along justification edges to depth-0 premises. It returns the set of assumptions underneath a conclusion, so you know *what to retract* if the conclusion must go.

- `test_premise_traces_to_itself` — base case
- `test_chain_traces_to_root` — A→B→C ⇒ `[a]`
- `test_diamond_traces_to_root` — diamond converges to single root
- `test_multiple_premises` — A,B→C ⇒ `{a, b}`
- `test_deep_chain_multiple_roots` — separate chains combine

Note it *returns premises, not derived nodes*. This is the "assumption-based" view of the justification graph — derived nodes are not the unit of retraction; only what the user has actually asserted can be retracted.

### 4b. `find_culprits(nogood_nodes) → ranked list`
Given the nodes in an active nogood, find which **premises** are responsible, ranked so the least disruptive retraction is first.

- Non-premise members are traced back to their premises (`test_derived_nodes_trace_to_premise` — nogood(B, C) with C←A yields `{a, b}`)
- Sorting is by **number of dependents ascending** — retract the offending premise that *fewest other beliefs depend on* (`test_sorted_by_dependents`)
- Each culprit reports `would_resolve` — the nogood members it would successfully knock out if retracted (`test_would_resolve_field`)

### 4c. `add_nogood(nodes) → changed`
This is the actual backtracking action, called by `Network.add_nogood` (the API layer exposes it as `api.add_nogood`, and `test_nogood_reports_backtrack` asserts the result has `backtracked_to`). The loop:

1. If any member already OUT → nothing to do.
2. `trace_assumptions` + `find_culprits` to rank premises.
3. **Pick the least-entrenched culprit** (see 4d) — not an arbitrary node.
4. `retract(culprit)` and **cascade is automatic**: all dependents whose justifications lose this support go OUT, recursively.
5. Log the culprit: `net.log` gets an entry with `action == "backtrack"` whose `value` contains `"culprit"` (`test_backtracking_logs_culprit`).
6. Return the list of truth-value changes.

The tests make the **dependency-directed** part explicit:
- `test_retracts_premise_not_derived`: nogood(B, C) with C←A retracts B (premise) — C stays IN because A still supports it. This is the crucial anti-naive behavior: *do not retract the derived conclusion, retract a premise that makes it go away*.
- `test_cascade_from_retracted_premise`: retracting premise A cascades B and C OUT.
- `test_shared_premise_resolves_both`: a single shared premise A (feeding B and C) is retracted once and kills both members of the nogood in one shot — the best possible resolution.

### 4d. Entrenchment — the tie-breaker for *which* culprit
The tests in `TestEntrenchment` show that "number of dependents" is not the only ordering input; there's an `_entrenchment(node_id)` score. Evidence in the tests:

| Factor | Effect | Test |
|---|---|---|
| `source` present on a premise | ↑ entrenched (protects evidence) | `test_evidence_premise_more_entrenched_than_derived`, `test_sourced_premise_over_unsourced` |
| `metadata["beliefs_type"] == "OBSERVATION"` | ↑ vs `"DERIVED"` | `test_beliefs_type_affects_entrenchment` |
| Number of dependents | ↑ entrenched | `test_more_dependents_more_entrenched` |
| Premise vs derived | premises score higher | `test_evidence_premise_more_entrenched_than_derived` |

The motivation is captured by `test_nogood_retracts_speculation_not_evidence`: when a simulation result contradicts a speculative interpretation built on an unsourced premise, the *speculation's premise* dies, and the evidence survives. This is the "physics-speculation bug" the test names — entropy that lets the system distrust conclusions-woven-from-air rather than observations. Note this is an *opinionated* extension to pure Doyle RMS (which only orders by assumptions); it bakes in an epistemics heuristic.

## 5. Propagation and logging

`api.propagate(db_path=DEFAULT_DB, ...)` (`reasons/api.py:773`) is the public entrypoint, wrapping whichever of `assert_node` / `retract` / `add_nogood` is being driven. The storage docstring (`reasons/storage.py`, module header) states the invariant:

> "ACID transactions ensure propagation cascades are atomic."

Every change is written to `propagation_log` (`reasons/storage.py`, `SCHEMA`): columns are `timestamp`, `action` (e.g. `"assert"`, `"retract"`, `"backtrack"`), `target`, `value`. The `add_nogood` log entry that appears in `test_backtracking_logs_culprit` is exactly this persistent record. When you run `api.explain`, that's the data it reads back — including `truth_value`, `outlist`, `violated_outlist` and (in the derived-node case) `antecedents`.

## 6. Persistence mapping

From `reasons/storage.py`:

- **Nodes** — one row per belief. `truth_value` (IN/OUT), `supporting_justification` (which justification is currently active; used by `_get_depth` in derive.py to pick the single active chain), `source`, `source_url`, `source_hash`, `text_hash`, and timestamps (`created_at`, `retracted_at`, etc.).
- **Justifications** — one row per SL rule. `antecedents_json` and `outlist_json` are the two node lists; `type` is `"SL"` today.
- **Nogoods** — one row per declared contradiction; `nodes_json` holds the ID list.
- **propagation_log** — append-only history of every truth-value change.

`save()` clears and rewrites, but carefully re-reads `node_tags`/`node_sources` first so externally-added metadata isn't lost. `load()` reconstructs and calls `network._rebuild_dependents()` to re-establish the dependent → antecedents index that backtracking traversal relies on.

## 7. What the observations *don't* show (and where the real code is)

The symbol search didn't find a `backtrack` symbol in `reasons/` because there is no such function — backtracking is implemented **inside `Network.add_nogood`** as driven by `find_culprits` + `trace_assumptions`. Likewise `network.py` is *imported* by `test_backtracking.py` and used everywhere, but its source isn't in the provided slices. The missing pieces needed to fully understand propagation mechanics:

- `reasons/network.py` — the `Network` class (assert/retract/explain/trace_assumptions/find_culprits/add_nogood/_entrenchment/_rebuild_dependents, and the cascade loop).
- The body of `reasons/api.py:773 propagate()` — what it drives and returns.
- Where `Node`, `Justification`, `Nogood` dataclasses are defined (imported from `reasons` package in `storage.py`).
- Whether `api.propagate` is a distinct cascade operation or just a name for "recompute after a mutation" (the test suite exercises cascades only through `assert_node`/`retract`/`add_nogood`, never via `propagate` directly — worth confirming).

Also note: the `find_rms_terms_in_reasons` observation reported `total_count: 0` for the pattern `outlist|nogood|antecedent|justification|backtrack`, but `reasons/derive.py` in the same snapshot clearly *contains* those strings (they appear in the `DERIVE_PROMPT`). That contradiction suggests the term search was scoped narrowly or misconfigured; I've gone with the direct code evidence from `derive.py`, `storage.py`, and the tests instead.

---

## Topics to Explore

- [file] `reasons/network.py` — The `Network` class is where the actual cascade loop, `assert_node`/`retract`, `trace_assumptions`, `find_culprits`, `_entrenchment`, `_rebuild_dependents`, and `add_nogood` live; none of these bodies are in the sliced observations.
- [function] `reasons/network.py:Network.add_nogood` — The concrete dependency-directed backtracking loop: which culprit it picks, how it invokes `retract`, and how it logs `action="backtrack"`.
- [function] `reasons/network.py:Network.find_culprits` — How premise ranking is computed, how `would_resolve` is derived, and where entrenchment enters the tie-break.
- [function] `reasons/api.py:propagate` — The `propagate` entrypoint at line 773; document whether it's a standalone recompute or a wrapper around mutations, and its returned payload.
- [general] `rms-model-definitions` — Locate the `Node`, `Justification`, and `Nogood` dataclasses (imported by `storage.py` from the `reasons` package root) to confirm the field-level API and default truth-value behavior.

## Beliefs

- `sl-justification-requires-all-antecedents-in-and-all-outlist-out` — An SL justification is only valid when every antecedent is IN and every outlist node is OUT; a node is IN iff at least one of its justifications is valid.
- `absent-outlist-node-is-treated-as-out` — An outlist node that does not exist in the network is treated as OUT (unbelieved), so default beliefs hold until the counter-evidence is explicitly asserted.
- `backtracking-retracts-premises-not-derived-nodes` — `add_nogood` retracts a depth-0 premise selected by `find_culprits`, never a derived node, and lets the change cascade OUT to dependents.
- `inactive-nogood-short-circuits-backtracking` — If any node in a nogood is already OUT, `add_nogood` returns an empty changed list and writes no `backtrack` entries to `propagation_log`.
- `entrenchment-prefers-evidence-over-speculation` — `_entrenchment` scores sourced premises, OBSERVATION-typed nodes, and high-dependency nodes above unsourced/derived/speculative ones, and is used to break culprit ties.
- `nogood-ids-are-content-derived-and-sorted` — Nogood IDs are formed from sorted node IDs joined by dashes (`nogood-a-b`), with numeric suffixes for duplicates, and are preserved verbatim through JSON/beliefs import.
