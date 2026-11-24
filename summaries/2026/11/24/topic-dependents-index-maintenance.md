---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Explore every code path that adds, removes, rebuilds, or manually adjusts `dependents` sets.

# Exploring Every Code Path That Mutates `dependents`

## ⚠️ First, a data-quality caveat

The JSON observations claim **0 occurrences** of `dependents`, **0** mutation calls (`.add/.remove/.discard/.update/.clear`), **0** assignments, and **no** definitions for `_rebuild_dependents`, `add_dependent`, or `remove_dependent`. This is clearly a false-negative from the scanner:

- The two attached test files alone contain dozens of `dependents` references (e.g. `net.nodes["a"].dependents.add("ghost")` at `tests/test_dependents_integrity.py` ~line 118, and throughout `tests/test_dangling_dependents.py`).
- The tests explicitly call `net._rebuild_dependents()` and expect `net.verify_dependents()` to exist.

**What is missing:** the actual implementation file (`reasons/network.py`, plus `reasons/__init__.py` for `Node.dependents = set()` and `Node.justification`). The scanner almost certainly missed it (maybe only non-test source directories weren't indexed, or the file pattern excluded `*.py` in `reasons/`). Everything below is reconstructed from the **contract expressed by the tests**, which is nonetheless quite prescriptive.

---

## Model: what `dependents` is

From the tests we can pin down the semantics precisely:

- `Node.dependents` is a **`set[str]`** of node ids — one per node.
- The invariant enforced by `verify_dependents()` is:
  > For every node `N`, and every justification `J` of `N`, and every id `x` in `J.antecedents ∪ J.outlist`, we must have `N ∈ node(x).dependents`.
  > And the reverse: every id in `node(x).dependents` must have a matching justification pointing at it.
- It is a **reverse index** of the justification graph (`antecedents ∪ outlist` on the forward side, `dependents` on the reverse side).
- Premises have an empty `dependents` set (see `test_premises_only`, `tests/test_dependents_integrity.py` ~line 40).

The forward edge is described by `Justification(type=..., antecedents=[...], outlist=[...])`. Both lists contribute to `dependents` (see `test_outlist_dependency` and `test_both_antecedent_and_outlist`, ~lines 62–88).

---

## The full mutation surface (reconstructed from tests)

### 1. `Network._rebuild_dependents()` — the **canonical rebuild**

**Location:** `reasons/network.py` (implementation), exercised by `tests/test_dependents_integrity.py::TestRebuildDependents` (~lines 24–108 of that file).

**Contract:**

- **Clears every node's `dependents` set to `set()` first.** `test_rebuild_clears_stale_entries` (~line 96) seeds a fake `"phantom"` and confirms it is gone after rebuild.
- **Then walks every node's justifications and re-adds the reverse edges** for both `antecedents` and `outlist`.
- Must be idempotent and safe on an empty network (`test_empty_network`).
- Result is verified clean by `verify_dependents()`.

This is the **only** path that is allowed to fully trash-and-rebuild. Any manual mutation elsewhere is expected to leave the index consistent *without* calling this (because the mutation tests call `assert_clean` immediately after each mutation).

### 2. `Network.verify_dependents()` — **read-only** consistency check

**Location:** `reasons/network.py::verify_dependents`, tested in `tests/test_dependents_integrity.py::TestVerifyDependents` (~lines 111–145).

- Returns a **list of error strings** (`[]` when clean).
- Distinguishes two error kinds: `"extra"` (dependent id present but no justification references it — `test_detects_extra_dependent`) and `"missing"` (justification references an antecedent/outlist id but the reverse edge is absent — `test_detects_missing_dependent`).
- **Explicitly does not mutate state.** `test_does_not_modify_state` (~line 138) inserts `"ghost"` and then asserts it still exists after `verify_dependents()` returns. This is important: there is **no** `verify_and_fix()` behavior hidden in here — a caller must call `_rebuild_dependents()` separately.

### 3. `Network.add_node(id, text, justifications=[...])` — populate on insert

**Test:** `test_after_add_node_with_justification` (~line 152).

When a node is added **with justifications**, every antecedent/outlist id must get the new node added to its `dependents`. This is where the initial reverse index is built on creation. Premises (`justifications=[]` or absent) contribute nothing.

The exact implementation is presumably a private helper (`_link_dependents(node)`) called both from `add_node` and `add_justification`, but we can't see it. **That helper is the single most important function to verify** — see the "Topics to Explore" section.

### 4. `Network.add_justification(id, justification)` — **add** a dependent edge

**Test:** `test_after_add_justification` (~line 165).

> After `net.add_justification("c", Justification(type="SL", antecedents=["b"]))`, the assertion `"c" in net.nodes["b"].dependents` holds.

So this path **adds** `c` to `dependents` for every id in the new justification's `antecedents` **and** `outlist`. Note: there is no dedicated `network.add_dependent(...)` API exposed by the tests (matching the empty `add_dependent_symbol` observation); the reverse edges are maintained by the network itself as a side-effect of forward-graph changes.

### 5. `Network.convert_to_premise(id)` — **remove** dependent edges

**Test:** `test_after_convert_to_premise` (~line 194).

> After converting derived `b` (whose only justification referenced `a`) to a premise, `"b" not in net.nodes["a"].dependents`.

Converting a derived node to a premise **discards its justifications**, therefore its reverse edges must be **removed from every antecedent/outlist node**. This is (so far) the only explicit *removal* path exercised by the visible tests. It is the mirror of `add_justification`.

**Implied helper:** something like `_unlink_dependents(node)` that walks `node.justification` (before discarding it) and does `node(ant).dependents.discard(this_id)`.

### 6. `Network.supersede(old_id, new_id)` — **rewrites** reverse edges

**Test:** `test_after_supersede` (~line 178).

> `"old" in net.nodes["new"].dependents`

Semantics: after superseding `old` by `new`, `old` is retained as a **justification on** the superseding node (typically as `outlist` or a negative antecedent — the classic "unless `new`, hold `old`" defeasible structure). So `new` must gain `old` in its `dependents` set.

This path may or may not remove pre-existing edges from `old`'s justifications — the visible test only asserts the **add** direction. The truncated tail of the file (lines 200–377) likely covers removal on supersede chains.

### 7. `Network.challenge(target, text)` — **adds** a dependent

**Test:** `test_after_challenge` (~line 184).

> `"target" in net.nodes[challenge_id].dependents`

The challenge creates a new node (the challenge) whose justification references `target` (likely via `outlist`, since a challenge is "unless target, hold the challenge"). That makes `target` a dependent of the challenge node.

This is important for the **propagation** side: retracting the target must cascade to the challenge (see `_propagate` below). And because `challenge()` itself triggers `_propagate` (per `test_challenge_with_dangling`), the mutation may **also** trigger `_propagate` — coupling the dependents mutation to truth-value updates.

### 8. `Network.defend(target, challenge_id, text)` — indirect

**Test:** `test_after_challenge_and_defend` (~line 191).

No direct `dependents` assertion; the test only calls `assert_clean` at the end. Defense most likely adds a justification to `target` citing the defense node, or vice-versa. The invariant is simply that any constraint added by defense leaves `verify_dependents() == []`.

### 9. `Network.add_nogood(...)` — **not shown fully**

`test_after_add_nogood` (~line 199) is cut off at line 200 of the returned content. The nogood's structural fields (likely `antecedents` listing nodes that cannot all be true) must produce reverse edges on each listed node. **This path is under-verified in the visible data** — see "Topics to Explore".

### 10. `Network.retract(id)` / `Network.assert_node(id)` — **do NOT mutate** `dependents` (only `_propagate` reads them)

**Tests:** `test_after_retract_and_restore` (~line 159); the entire `tests/test_dangling_dependents.py`.

These paths flip `truth_value` and call `_propagate()`, which **reads** `node.dependents` to cascade. They do **not** add or remove reverse edges — the graph is unchanged, only truth values. This matters because tests inject ghosts into `dependents` and then call `retract`/`assert_node` — the ghosts must persist as `dependents` (they do), and `_propagate` must simply *skip* them with a warning.

### 11. `_propagate()` — **reader**, plus a recursive-mutation safeguard

**Location:** `reasons/network.py::_propagate` (per the docstring of `tests/test_dangling_dependents.py` — "dangling dependent guard in `_propagate` (issue #22)").

Contract from `TestDanglingDependentNoCrash`, `TestDanglingDependentWarningLogs`, etc.:

- Iterates `node.dependents`; for each id not in `net.nodes`, emits a warning log entry of shape `{"action": "warn", "target": <ghost-id>, "value": "...dangling...", "timestamp": ...}` and **continues** instead of raising `KeyError`.
- Ghost ids never appear in the returned `changed` list (`test_dangling_not_in_changed_list`, `test_diamond_with_dangling`).
- Valid dependents continue to propagate past ghosts (`test_valid_dependent_still_retracted`, `test_chain_with_dangling_in_middle`).

`_propagate` is thus the **only consumer** that depends on the invariant being correct — a corrupted `dependents` set doesn't crash, it just produces warnings. That's why `verify_dependents()` exists as a separate, explicit integrity surface.

### 12. Manual adjustment — **tests only**

**Test:** every `net.nodes["a"].dependents.add(...)` / `.update(...)` / `.discard(...)` in the two test files, e.g. `tests/test_dependents_integrity.py::test_rebuild_clears_stale_entries` (~line 100), `test_detects_missing_dependent` (~line 130), and all of `tests/test_dangling_dependents.py`.

There is **no production API** for directly mutating a `dependents` set; production callers should not touch it — the tests deliberately do so to *simulate corruption* and prove that `verify`/`rebuild`/`_propagate` tolerate it. If you find production code doing `node.dependents.add(...)` outside a `_link_dependents` helper, that's a code smell: it violates the invariant unless accompanied by a matching justification mutation.

---

## Summary table

| Path | What it does to `dependents` | Test |
|---|---|---|
| `add_node(justifications=...)` | **adds** reverse edges to each `antecedents`/`outlist` id | `test_after_add_node_with_justification` |
| `add_justification(id, j)` | **adds** `id` to every `j.antecedents`/`j.outlist` node | `test_after_add_justification` |
| `convert_to_premise(id)` | **removes** `id` from its former justifications' nodes | `test_after_convert_to_premise` |
| `supersede(old, new)` | **adds** `old` to `new.dependents` (and possibly removes others) | `test_after_supersede` |
| `challenge(target, text)` | **adds** `target` to `challenge_id.dependents` | `test_after_challenge` |
| `defend(...)` | indirect — must leave index clean | `test_after_challenge_and_defend` |
| `add_nogood(...)` | partial — see truncated tail | `test_after_add_nogood` |
| `_rebuild_dependents()` | **clears then rebuilds** every node's set | `TestRebuildDependents` |
| `verify_dependents()` | **read-only** | `TestVerifyDependents` |
| `_propagate()` | **reads**, tolerates dangling, logs warnings | `TestDanglingDependent*` |
| `retract` / `assert_node` | **no change to edges**, propagate only | `test_after_retract_and_restore` |

---

## What is still unknown (and matters)

1. **The actual definition site**: `Node.dependents` initializer, and the `_link_dependents` / `_unlink_dependents` helpers (or whatever they're named). The scanner returned 0 matches, so `reasons/network.py` and `reasons/__init__.py` were **not captured** by the observation.
2. **Lines 200–377 of `tests/test_dependents_integrity.py`** — includes the rest of `add_nogood` coverage, plus (per the file docstring) tests of "corruption detection" and "rebuild repairs it" in scenarios beyond the four shown (likely supersede chains, convert-on-chain, add_nogood, delete/ unassert paths).
3. **Lines 200–223 of `tests/test_dangling_dependents.py`** — the tail covers `add_justification` with dangling refs (`test_add_justification_with_dangling` is truncated mid-setup).
4. **Any `remove_node` / `delete` path** — not exercised by the visible tests, but if such a method exists, it must either cascade-remove ids from every antecedent's `dependents` or leave stale refs for `verify` to flag. **This is the biggest gap.**
5. **Consistency-on-failure**: if `add_justification` validates the justification *after* linking dependents and then raises, does it roll back the link? No test shown covers failure paths.

## Topics to Explore

- [file] `reasons/network.py` — The actual implementation of `Network`, `_rebuild_dependents`, `verify_dependents`, `_propagate`, and the private link/unlink helpers that maintain the `dependents` invariant; the JSON scanner missed this file entirely, so it must be read directly.
- [function] `reasons/network.py:_rebuild_dependents` — Canonical rebuild; confirm it clears first, walks both `antecedents` and `outlist`, and is idempotent.
- [function] `reasons/network.py:convert_to_premise` — The only proven *removal* path in the visible tests; verify it discards prior justifications and unlinks reverse edges before replacing them with a premise justification.
- [file] `reasons/__init__.py` — Where `Node` is defined; confirm `dependents` is initialized as a fresh `set()` per node (shared mutable default would be catastrophic).
- [general] `network-remove-node-path` — Whether `Network` exposes a `remove_node` / `delete` and, if so, how it retires the node's reverse edges; not covered by the visible tests, but a likely integrity hazard.

## Beliefs

- `dependents-is-reverse-index-of-justifications` — `node(id).dependents` is exactly the set of node ids whose any justification lists `id` in either `antecedents` or `outlist`; `verify_dependents()` enforces both directions of this bijection.
- `verify-dependents-is-read-only` — `Network.verify_dependents()` returns a list of error strings and never mutates state; repair requires a separate call to `_rebuild_dependents()`.
- `rebuild-dependents-clears-before-rebuilding` — `_rebuild_dependents()` empties every node's `dependents` set before re-populating it from justifications, so it removes stale entries unconditionally.
- `propagate-tolerates-dangling-dependents` — `_propagate()` skips ids in `dependents` that are not in `net.nodes`, logs a `"warn"` entry with `target=<ghost>` and a `"dangling"` message, and never raises `KeyError`.
- `retract-and-assert-do-not-mutate-graph` — `retract()` and `assert_node()` change only `truth_value` and run `_propagate()`, they do not add or remove any edges from `dependents`.
- `convert-to-premise-removes-dependent-edges` — `convert_to_premise(id)` removes `id` from every former antecedent's `dependents` set as part of discarding its justifications.
