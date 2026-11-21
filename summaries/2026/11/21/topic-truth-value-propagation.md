---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: how `IN`/`OUT`/`UNKNOWN` is recomputed across dependents during any mutation, which underpins the cascade tests.

## How `IN`/`OUT`/`UNKNOWN` Is Recomputed Across Dependents

### Important caveat up front

The observations do **not** include the implementation of the recompute loop (`Network._recompute_status` / `_propagate` / whatever it's called), the `Node` dataclass, or where the value `UNKNOWN` is defined. All grep patterns (`UNKNOWN`, `dependents`, `recomput|cascade|...`, and the cascade test pattern) returned **zero matches**, which is inconsistent with the test files that clearly contain `dependents` — so the pattern searches appear to have failed or run against a narrower scope than promised.

What *is* observable is:

- The **storage schema** and `Storage.save()` in `reasons/storage.py` (first ~250 lines).
- **Test files** that define the expected behavior of the reverse index (`tests/test_dependents_integrity.py`, lines ~1–250) and of backtracking cascades (`tests/test_backtracking.py`, lines ~1–250).
- The derive-prompt scaffolding in `reasons/derive.py`, which describes the semantics ("cascade OUT automatically", "flips IN when it goes OUT", ALL vs ANY) in prose only.

So this answer reconstructs the mechanism from the **contract the tests enforce**, and flags what remains unverified.

---

### 1. The reverse index: `Node.dependents`

Each node carries a `dependents: set[str]` (visible in `tests/test_dependents_integrity.py`, e.g. `net.nodes["a"].dependents == {"b"}` at the `test_single_dependency` case). This is a reverse map: `x.dependents` = the set of nodes whose truth value may change when `x`'s truth value changes.

From the tests, membership is defined over **all** graph references, not just antecedents:

| Reference kind | Test |
|---|---|
| Antecedent (`SL` `antecedents=[…]`) | `test_single_dependency` |
| `outlist` members (negative-support), even with no antecedents | `test_outlist_dependency` |
| Both antecedents and outlist on the same justification | `test_both_antecedent_and_outlist` |
| `supersede` link (new node's `dependents` gains the old node) | `test_after_supersede` |
| `challenge` (challenge node's `dependents` gains the target) | `test_after_challenge` |
| `summarize` (`over=[…]` produces a summary node whose antecedent set is those nodes) | `test_after_summarize` |

`_rebuild_dependents()` is the canonical reconstruction: it **clears** every node's `dependents` (`test_rebuild_clears_stale_entries` shows a phantom entry surviving until this call), then walks every justification of every node and inserts the declaring node into each referenced node's set. It is idempotent (`test_rebuild_idempotent`). `verify_dependents()` is a **read-only** audit that returns a list of error strings — `"extra …"` for stale entries (`test_detects_extra_dependent`) and `"missing …"` for absent ones (`test_detects_missing_dependent`) — and explicitly does not mutate state (`test_does_not_modify_state`).

### 2. What the mutation contract requires

`TestMutationPathIntegrity` (in `tests/test_dependents_integrity.py`) pins one invariant: every mutation path in `Network` leaves `verify_dependents() == []`. Enumerated paths:

1. `add_node(..., justifications=[…])`
2. `retract(nid)` / `assert_node(nid)`
3. `add_justification(nid, …)`
4. `supersede(old, new)`
5. `challenge(target, …)` / `defend(target, ch, …)`
6. `convert_to_premise(b)` — **removes** `b` from `a.dependents` (it strips justifications)
7. `add_nogood([…])`
8. `summarize(summary, text, over=[…])`

Two of these directly give us the traversal shape:

- `test_after_retract_and_restore` — retracting `a` (which has dependent `b`) then re-asserting it must leave the index clean in both directions. This implies the recompute walks **outward** through `dependents` when a node's value flips, so that `b`'s value is re-derived from `a`'s new state.
- `test_after_add_justification` — adding a second `SL` justification to `c` (now supported by both `a` and `b`) adds `c` to `net.nodes["b"].dependents`. So **every** justification's antecedents/outlist are indexed, not just the selected one (`supporting_justification`).

### 3. How the recompute loop must work (inferred)

Recovering the algorithm from the tests plus the storage schema:

- **Truth values live on `nodes.truth_value`** (`reasons/storage.py` line ~21: `truth_value TEXT NOT NULL DEFAULT 'IN'`). A freshly added premise is therefore `IN`.
- `retract(nid)` sets it to `OUT`; `assert_node(nid)` sets it back to `IN` (`test_after_retract_and_restore`).
- Justification selection is tracked in `nodes.supporting_justification` (INTEGER index into the node's justification list, `reasons/storage.py` line ~22). `reasons/derive.py` (`_get_depth`) confirms the interpretation: when `supporting_justification` is set, only that one justification is considered "the active rule"; otherwise all are considered.
- `justifications.antecedents_json` and `justifications.outlist_json` (schema, lines ~30–31) hold the two reference kinds. Outlist is the GATE construct described in `DERIVE_PROMPT`: a GATE node is `IN` iff its antecedents are `IN` **and** the outlist nodes are `OUT`.
- `propagation_log` (`timestamp, action, target, value`, lines ~37–42) is what `Network.log` mirrors — `test_backtracking_logs_culprit` asserts an entry with `action == "backtrack"` and `"culprit"` in `value`.

Reconstruction of the expected loop (label this **inferred**, not from source):

```
mutate(nid, change):
    apply(change)                                   # retract/assert/add_j/…
    frontier = {nid}
    if apply changed damper chain:
        queue = nid.dependents ∪ {nid_direct_descendants}
    while queue:
        for n in queue:
            old = n.truth_value
            n.truth_value = _evaluate(n)            # see below
            if n.truth_value != old:
                queue |= n.dependents               # cascade
    # log every flip to propagation_log / net.log
```

`_evaluate(n)` (inferred):

- If `n` has no justifications → it is a **premise**; leave as `IN` unless explicitly retracted.
- Else, for each justification (or just `supporting_justification` if set):
  - `SL` valid iff `all(a IN for a in antecedents)` and `all(o OUT for o in outlist)`.
- `n` is `IN` iff any justification is valid; otherwise `OUT`.

The "cascade" tests pin exactly this:

- `test_cascade_from_retracted_premise` — `A → B → C`, `Nogood(C, D)`. `D` is chosen for retraction (fewest dependents). Since `D`'s flip touches zero dependents, nothing cascades. The test asserts `B` and `C` stay `IN`, confirming retracting `D` doesn't wander into `A`'s subtree.
- `test_shared_premise_resolves_both` — `A → B` and `A → C`, `Nogood(B, C)`. Retracting `A` flips its `dependents = {B, C}`, both to `OUT`. This is the cascade with a branching fan-out.
- `test_retracts_premise_not_derived` — with `A → C, B`, `Nogood(B, C)`, `B` is retracted (fewer dependents than `A`) and `C` stays `IN` because `A` is untouched. Demonstrates the "retract, then recompute outward" flow doesn't over-retract.

### 4. `add_nogood` — the mutation that triggers cascades most aggressively

From `tests/test_backtracking.py`:

- `trace_assumptions(nid)` walks back through **justifications** to reach depth-0 nodes (`TestTraceAssumptions`, especially `test_diamond_traces_to_root` and `test_deep_chain_multiple_roots`). This is precisely the reverse direction of `dependents`: forward is `x.dependents → …`, backward is `justification.antecedents → premise`.
- `find_culprits(nodes)` returns the premise-level culprits, sorted by dependent count (`test_sorted_by_dependents`: criminal with fewer dependents comes first) and annotated with `would_resolve` — the derived nodes that would fall `OUT` if the culprit were retracted.
- `add_nogood([...])` then calls `retract()` on the chosen culprit; that retraction kicks off the cascade loop described above. `test_backtracking_logs_culprit` asserts a `"backtrack"` entry is emitted before the cascade log entries.

So `add_nogood` is not a separate recompute path: it *composes* with `retract` (plus `find_culprits` for victim selection). The cascade behavior of `retract` is what `TestCascadeFromRetractedPremise` and `TestSharedPremiseResolvesBoth` are really testing.

### 5. Atomicity

`Storage` opens the SQLite connection with `PRAGMA journal_mode=WAL` and `PRAGMA foreign_keys=ON`, and `Storage.save` runs inside `with self.conn:` — the docstring at `reasons/storage.py` line ~4 states *"ACID transactions ensure propagation cascades are atomic."* In practice, `Network` is the in-memory source of truth and `save()` is an all-or-nothing snapshot; a mid-cascade failure leaves only the DB's prior snapshot visible.

---

### 6. What is missing (and would need to be added to fully answer)

- `reasons/network.py` — the actual `Network` class. The tests point at `Network._rebuild_dependents`, `verify_dependents`, `trace_assumptions`, `find_culprits`, `_entrenchment`, `add_node`, `retract`, `assert_node`, `add_justification`, `supersede`, `challenge`, `defend`, `convert_to_premise`, `add_nogood`, `summarize`. None of these implementations were returned.
- The `Node` dataclass definition (where `dependents: set[str]` and `truth_value` are declared, and whether `truth_value` defaults to `"IN"` in Python or only in SQL).
- Where `UNKNOWN` actually appears. The grep for `"UNKNOWN"` returned 0 matches; `tests/test_dependents_integrity.py` and `tests/test_backtracking.py` only assert `IN`/`OUT`. `UNKNOWN` may be a legacy label, or live in `network.py` outside the search scope. **Should be verified before documenting it as a first-class status.**
- The implementation of `retract`, `trace_assumptions`, `find_culprits`, and the "cascade" queue mechanics (depth-first vs breadth-first, cycle guard behavior, whether flips are logged per-node).
- The remaining 232 lines of `tests/test_dependents_integrity.py` (line 251+) and 60 lines of `tests/test_backtracking.py` (line 251+) — the tail of `TestEntrenchment` and any additional cascade assertions were truncated.

## Topics to Explore

- [function] `reasons/network.py:Network.retract` — The mutation that seeds the cascade; every nogood resolution and every `TestMutationPathIntegrity` case funnels through it.
- [function] `reasons/network.py:Network._rebuild_dependents` — Canonical reverse-index rebuild; confirms whether the loop is a simple "iterate justifications, insert into antecedent.dependents" or something richer.
- [function] `reasons/network.py:Network.find_culprits` — How `would_resolve` and dependent-count sorting choose the retraction victim, which controls whether a cascade fires at all.
- [file] `reasons/node.py` (or wherever `Node` lives) — The `Node` dataclass declaration of `truth_value` and `dependents`; needed to answer whether `UNKNOWN` exists as a first-class state.
- [general] `truth-value-semantics` — Whether `UNKNOWN` is a distinct third state in this codebase or absent (grep for `"UNKNOWN"` returned 0 matches), and if absent, whether it should be reintroduced for "no justification yet" nodes.

## Beliefs

- `dependents-index-is-a-reverse-index` — Each `Node.dependents` set contains the nodes that list it as an antecedent **or** an outlist member, so outlist-only GATE nodes are indexed too (`test_outlist_dependency`, `test_both_antecedent_and_outlist`).
- `rebuild-dependents-is-canonical-repair` — `_rebuild_dependents()` clears the index and reconstructs it from justifications; `verify_dependents()` reports drift without mutating state (`test_detects_extra_dependent`, `test_does_not_modify_state`).
- `every-mutation-path-preserves-dependents-invariants` — `add_node`, `retract`/`assert_node`, `add_justification`, `supersede`, `challenge`/`defend`, `convert_to_premise`, `add_nogood`, and `summarize` must each leave `verify_dependents() == []` (`TestMutationPathIntegrity`).
- `nogood-backtracking-retracts-fewest-dependents-culprit` — `add_nogood` traces each nogood node back to premises via `trace_assumptions`, then retracts the culprit with the fewest dependents, which is what dictates the cascade size (`test_sorted_by_dependents`, `test_shared_premise_resolves_both`).
- `truth-value-column-defaults-to-in` — The `nodes.truth_value` SQL column defaults to `'IN'` (`reasons/storage.py` schema), so a premise added without justifications is IN until explicitly retracted (`test_after_retract_and_restore`).
