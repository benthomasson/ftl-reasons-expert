---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: _rebuild_dependents in reasons/network.py

# `_rebuild_dependents` — Dependency Reverse-Index Rebuilder

## Purpose

`_rebuild_dependents` reconstructs the **reverse dependency index** (`Node.dependents`) from scratch by walking every node's justifications. In this TMS (Doyle's algorithm), truth values propagate *forward* from a changed node to everything that references it — which means the network needs an efficient map of "who points at me?" rather than scanning every justification on every propagation step.

Every justification expresses two ways a node can influence another:

- **`antecedents` (inlist)** — the node must be `IN` for the dependent's justification to be valid.
- **`outlist`** — the node must be `OUT` for the dependent's justification to be valid.

Both directions matter for truth-value changes, so both are recorded in `dependents`. This function is the **canonical way to rebuild that index** when it might have drifted (e.g., after deserialization, hand-editing, or a suspected bug). The docstring explicitly says all other rebuild call sites must delegate here.

## Contract

**Preconditions**
- `self.nodes` is a `dict[str, Node]` whose values expose `.justifications`, `.dependents`, `.id`.
- Each `Justification` exposes `.antecedents` and `.outlist` as iterables of node-ID strings.
- `Node.dependents` is assignable (a `set` attribute).
- Falsely assumed by callers but *not* type-enforced: `node.justifications` is never `None`. (It's treated as a plain iterable; `None` would raise `TypeError`.)

**Postconditions**
- Every node's `dependents` set reflects exactly the set of node IDs whose justifications reference it via `antecedents` or `outlist`, filtered to those present in `self.nodes`.
- Stale entries are gone (they are wiped in a first pass, not merely corrected).
- No dangling node IDs remain in any `dependents` set (they're skipped during rebuild, and existing dangling entries were already cleared).

**Invariants preserved (but not checked here)**
- The `dependents` index is a faithful *mirror* of the justification graph — this is exactly the condition `verify_dependents` asserts by recomputing the same mapping.

## Parameters

None. The function operates entirely on `self.nodes` and mutates the per-node `dependents` attribute.

## Return Value

Returns `None`. It is a pure side-effecting rebuild — callers get no summary, no changed/fixed list, and no signal about consistency (see Error Handling).

## Algorithm

```
1. For each node in self.nodes.values():
       node.dependents = set()          # wipe stale index

2. For each node in self.nodes.values():
       For each justification j in node.justifications:
           For each ant_id in j.antecedents:
               If ant_id in self.nodes:
                   self.nodes[ant_id].dependents.add(node.id)
           For each out_id in j.outlist:
               If out_id in self.nodes:
                   self.nodes[out_id].dependents.add(node.id)
```

The two passes are important:

- **Pass 1 (reset)** guarantees the postcondition regardless of the prior state. Without it, incremental-only callers (e.g., `add_node`, `add_justification`) would accumulate duplicates or stale edges from previously removed justifications.
- **Pass 2 (rebuild)** is essentially a graph-transposition: iterate every outgoing reference and append the *owner* to the *target's* `dependents`. Because `dependents` is a `set`, duplicate references (same node/justification) are deduped automatically.

Insertion order of `node.id` into sets does not matter for correctness; no ordering is produced.

## Side Effects

- Mutates `dependents` on **every** `Node` in `self.nodes` (all are reset, even if they have no references).
- No logging (`self._log` is not called), no I/O, no time-based updates (`updated_at`, `retracted_at`, etc. are untouched).
- Does **not** trigger propagation — it only repairs the index, so truth values are unchanged even if the rebuild reveals different edges than before.

## Error Handling

- **No exceptions are raised for dangling references.** Nodes referenced in `antecedents`/`outlist` that are not in `self.nodes` are silently skipped (`if ant_id in self.nodes` / `if out_id in self.nodes`). This means a corrupt/imported network with dangling refs can be repaired silently — the caller cannot distinguish "clean rebuild" from "rebuild that dropped dangling edges."
- **`node.justifications` iterated directly** — if a caller ever sets `justifications = None`, this raises `TypeError`. There is no guard for that.
- **Unknown attribute shapes** (e.g., a `Justification` without `antecedents`) also raise `AttributeError`; the function does not defend against malformed objects.

## Usage Patterns

Intended call sites:

- **After deserialization / import** where `dependents` may not have been persisted or may be stale.
- **Sanity repair** after operations that mutated justifications indirectly (e.g., scripts, migrations).
- **Before `verify_dependents()`** in tests to confirm the index can be reconstructed, then to confirm it matches.

Caller obligations:

- Do **not** rely on this to propagate truth values; call `recompute_all()` or `_propagate()` afterwards if truth values need to reflect the corrected graph.
- To preserve the "single canonical implementation" invariant, any *new* rebuild code should call this method rather than reimplement the traversal.

Sibling routines that maintain `dependents` **incrementally** (not via this function) include:
- `add_node` — adds new node to antecedents'/outlist's `dependents` on insert.
- `add_justification` — same, one justification at a time.
- `remove_justification` / `convert_to_premise` — discard edges that no longer appear in any remaining justification.
- `supersede` / `challenge` — add the target as a dependent after appending to the outlist.

Those incremental paths are the behavior this rebuild must reproduce; any divergence is a bug surfaced by `verify_dependents`.

## Dependencies

- `Node` and `Justification` from `reasons/__init__.py` (imported at the top).
- `Network.nodes`, `Network._...` — only `self.nodes` is touched.
- No external libraries.

## Topics to Explore

- [function] `reasons/network.py:verify_dependents` — The non-mutating counterpart that recomputes the same expected mapping and diffs it against live `dependents`; the natural test oracle for this rebuild.
- [file] `tests/test_dependents_integrity.py` — Presumably asserts that incremental mutations (add/remove justification, supersede, challenge) keep `dependents` consistent with `verify_dependents`; useful to see which edge cases are covered.
- [function] `reasons/network.py:_propagate` — The consumer of `dependents`; explains why the reverse index exists and what breaks if it's stale.
- [function] `reasons/network.py:add_justification` — Incremental counterpart whose edge-registration logic must match this rebuild's traversal exactly.
- [general] `tms-nonmonotonic-outlist-semantics` — The interplay of justification `antecedents` (IN to satisfy) vs `outlist` (OUT to satisfy) explains why *both* are treated as "dependents" here.

## Beliefs

- `rebuild-dependents-is-canonical` — The docstring designates `_rebuild_dependents` as the sole canonical rebuild; other rebuild sites should delegate to it.
- `outlist-nodes-are-dependents` — `dependents` contains IDs referenced through justification `outlist` in addition to `antecedents`.
- `dangling-references-are-silently-skipped` — Antecedent/outlist IDs not present in `self.nodes` are ignored rather than raising.
- `rebuild-clears-before-rebuilding` — A first pass resets every node's `dependents` to `set()` before edges are re-added, so stale entries cannot survive a rebuild.
- `self-referencing-justifications-produce-self-dependents` — A node whose justification lists itself in `antecedents` or `outlist` will have its own ID added to its own `dependents` set; no cycle filter exists.
- `rebuild-does-not-propagate-truth-values` — Calling `_rebuild_dependents` leaves all `truth_value` fields unchanged; propagation must be triggered separately.
