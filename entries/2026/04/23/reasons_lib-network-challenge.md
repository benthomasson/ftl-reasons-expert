# Function: challenge in reasons_lib/network.py

**Date:** 2026-04-23
**Time:** 16:37

# `Network.challenge()` — `reasons_lib/network.py`

## Purpose

`challenge` implements dialectical argumentation within a Doyle-style Truth Maintenance System. It lets you say "I disagree with node X, here's why" — and the network automatically retracts X (and anything downstream of X) through the standard propagation machinery.

It does this by creating a new **challenge node** (a premise, so IN by default) and wiring it into the target's **outlist**. Since SL justifications require all outlist nodes to be OUT, an IN challenge invalidates the target's justification, driving it OUT. This is the same outlist mechanism used by `supersede` — the difference is semantic, not mechanical.

The design is reversible: retracting or defending the challenge node makes it go OUT, which restores the target. This enables the full challenge → defend → counter-challenge dialectical cycle without losing any nodes.

## Contract

**Preconditions:**
- `target_id` must exist in `self.nodes` (enforced via `KeyError`)
- If `challenge_id` is provided, it must not already exist (enforced via `ValueError`)

**Postconditions:**
- A new premise node exists in the network with the challenge ID
- The challenge ID appears in every justification's outlist on the target (or a new SL justification was created for premise targets)
- The target's `metadata["challenges"]` list includes the new challenge ID
- The challenge node's `dependents` set includes `target_id`
- Truth values have been recomputed and propagated through all affected nodes

**Invariant:**
- The challenge node is always a premise (no justifications), so it is IN unless explicitly retracted

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `target_id` | `str` | ID of the node being challenged. Must exist in `self.nodes`. |
| `reason` | `str` | Human-readable explanation for the challenge. Stored as the challenge node's `text`. |
| `challenge_id` | `str \| None` | Optional explicit ID for the challenge node. If `None`, auto-generated as `challenge-{target_id}`, with a numeric suffix (`-2`, `-3`, ...) if that name is taken. If provided and already exists, raises `ValueError`. |

**Edge cases on `challenge_id`:** The auto-generation loop starts the suffix at 2 (not 1), so the sequence is `challenge-a`, `challenge-a-2`, `challenge-a-3`. A caller providing `challenge_id="challenge-a"` when that already exists will get a `ValueError` — the auto-dedup logic only runs when `challenge_id is None`.

## Return Value

```python
{"challenge_id": str, "target_id": str, "changed": list[str]}
```

- `challenge_id`: The ID of the newly created challenge node
- `target_id`: Echo of the input (useful when callers pass through results)
- `changed`: List of node IDs whose truth values changed as a result. The target itself is included if it changed. Downstream nodes that cascaded are appended via `_propagate`. **Empty list** if the target was already OUT (e.g., from a prior retraction or another challenge)

The caller should use `changed` to understand blast radius — which beliefs were affected.

## Algorithm

1. **Validate** the target exists.

2. **Generate challenge ID** — if not provided, try `challenge-{target_id}`, then append incrementing suffixes until unique. If explicitly provided, verify it's not taken.

3. **Create the challenge node** — a bare `Node` with no justifications (premise), so `truth_value` defaults to `"IN"`. Stores `{"challenge_target": target_id}` in metadata.

4. **Wire into the outlist** — this is the key step. Two cases:
   - **Target has justifications:** Append the challenge ID to the outlist of *every* existing justification. This means *all* justifications now require the challenge to be OUT. If the target had multiple justifications and you only added the challenge to one, the other justification could still keep it IN.
   - **Target is a premise** (no justifications): Convert it from an implicit premise to a justified node by creating an SL justification with no antecedents and the challenge in the outlist. Semantically: "believe this unless the challenge holds." Before conversion, the node was IN because premises default to IN; after, it's IN because the SL justification `(antecedents=[], outlist=[challenge])` is valid — but only while the challenge is OUT.

5. **Register dependency** — add `target_id` to the challenge node's `dependents` set so the target gets recomputed when the challenge changes (e.g., retraction or defense).

6. **Track metadata** — append the challenge ID to the target's `metadata["challenges"]` list.

7. **Recompute and propagate** — call `_compute_truth` on the target. If truth value changed (typically IN → OUT), record it and BFS-propagate through `_propagate`, which recomputes all downstream dependents.

## Side Effects

- **Mutates `self.nodes`**: Adds the challenge node, modifies the target's justifications (outlist mutation or new justification), updates metadata on both nodes
- **Mutates `self.log`**: Appends audit log entries for the add and challenge actions
- **Propagation cascade**: Can transitively change truth values of any node reachable through the `dependents` graph. A challenge to a foundational premise can flip an arbitrarily deep chain of derived nodes to OUT.

## Error Handling

| Exception | Condition |
|-----------|-----------|
| `KeyError` | `target_id` not found in `self.nodes` |
| `ValueError` | Explicit `challenge_id` already exists in the network |

No errors are swallowed. If the challenge doesn't change the target's truth value (target was already OUT), the method succeeds silently — the challenge node is still created and wired in, it just doesn't trigger propagation.

## Usage Patterns

**Basic challenge:**
```python
result = net.challenge("a", "A is wrong because X")
# result["changed"] includes "a" and any downstream nodes
```

**Challenge → defend cycle:**
```python
net.challenge("a", "A is wrong")         # a goes OUT
net.defend("a", "challenge-a", "No, A is right because Y")  # challenge-a goes OUT, a restored
```

**Dismissal via retraction:**
```python
net.challenge("a", "A is wrong")
net.retract("challenge-a")  # challenge goes OUT, a comes back IN
```

**Multi-level dialectics:** You can challenge a defense (which is itself a challenge to a challenge), building an argumentation tree of arbitrary depth. Each level uses the same `challenge` mechanism.

**Caller obligations:** The caller does not need to manually propagate or recompute anything — `challenge` handles all of it. The caller should inspect `changed` to understand what was affected.

## Dependencies

- **`Node`, `Justification`** from `reasons_lib/__init__.py` — the data model. `Node.dependents` is the reverse index enabling propagation; `Justification.outlist` is the mechanism that makes challenges work.
- **`_compute_truth`** — evaluates whether any justification is valid. A justification is valid iff all antecedents are IN *and* all outlist nodes are OUT.
- **`_justification_valid`** — the predicate `_compute_truth` uses. This is where the outlist semantics live.
- **`_propagate`** — BFS traversal of `dependents` graph to cascade truth value changes.
- **`_log`** — audit trail (no external I/O, just appends to `self.log`).

## Assumptions Not Enforced by the Type System

1. **`truth_value` is always `"IN"` or `"OUT"`** — it's typed as `str`, so nothing prevents other values.
2. **Node IDs are globally unique across the network** — enforced at runtime but not structurally.
3. **Justification type is `"SL"` or `"CP"`** — typed as `str`; `_justification_valid` silently returns `False` for unknown types.
4. **The challenge node is never given justifications** — it's created as a premise by construction, but nothing prevents a caller from later adding justifications to it via `add_justification`.
5. **`metadata["challenges"]` is a list** — the method does `get("challenges", [])` then appends, but if someone set it to a non-list, this would fail.

---

## Topics to Explore

- [function] `reasons_lib/network.py:defend` — The counterpart to challenge; uses `challenge` internally to attack the challenge node, creating the reversible dialectical cycle
- [function] `reasons_lib/network.py:_justification_valid` — The core predicate: understanding outlist semantics is essential to understanding why challenges work
- [function] `reasons_lib/network.py:supersede` — Uses the same outlist mechanism for a different semantic purpose (version replacement vs. argumentation)
- [file] `tests/test_dialectical.py` — Comprehensive test suite covering the full challenge/defend/counter-challenge lifecycle and persistence round-trips
- [general] `doyle-1979-tms` — Doyle's original Truth Maintenance System paper; the outlist/inlist terminology and SL justification semantics originate there

## Beliefs

- `challenge-uses-outlist-mechanism` — Challenge works by adding the challenge node to the target's outlist, reusing the same non-monotonic reasoning mechanism as supersede
- `challenge-converts-premises` — When a premise (node with no justifications) is challenged, it is silently converted to a justified node with an SL justification containing empty antecedents and the challenge in the outlist
- `challenge-node-is-premise` — The challenge node itself is always created as a premise (no justifications, IN by default) and can only be driven OUT by explicit retraction or by being challenged/defended
- `challenge-modifies-all-justifications` — When the target has multiple justifications, the challenge is added to the outlist of every one of them, ensuring no justification can independently keep the target IN
- `challenge-propagation-is-automatic` — After creating the challenge, truth value recomputation and BFS propagation through dependents happen within the same call; callers never need to trigger propagation manually

