---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: tests/test_dialectical.py

# `tests/test_dialectical.py`

## Purpose

This file validates the **dialectical argumentation** subsystem of the `reasons` knowledge network — a challenge/defend mechanism that lets you dispute and rebut beliefs rather than simply asserting or removing them. It's rooted in Doyle's Section 6 framework, where arguments become first-class, explicit, and *challengeable* beliefs in the network graph.

The file owns the contract for:
- Challenging a node (flipping its truth value to `OUT`)
- Defending against a challenge (neutralising the challenge, restoring the target)
- Multi-level dialectical chains (challenge → defense → counter-challenge → counter-defense)
- Cascade propagation of truth-value changes through dependency edges
- SQLite persistence round-trip of all dialectical state
- The public API surface (`reasons.api`) for challenge/defend operations

## Key Components

### `TestChallenge`
Tests the `Network.challenge(node_id, reason, challenge_id=None)` method.

| Test | Contract Verified |
|------|-------------------|
| `test_challenge_premise` | Challenging an `IN` node flips it to `OUT`; returns `{"challenge_id": "challenge-a", "changed": ["a"]}` |
| `test_challenge_derived` | Works on nodes with `Justification` antecedents, not just bare premises |
| `test_challenge_cascades` | Downstream dependents (via `SL` justification) also flip to `OUT` and appear in `result["changed"]` |
| `test_challenge_creates_node` | A new node `challenge-a` is created with the reason text and truth value `IN` |
| `test_challenge_metadata` | Bidirectional metadata links: `challenge-a.metadata["challenge_target"] == "a"` and `"a"` appears in `a.metadata["challenges"]` |
| `test_custom_challenge_id` | `challenge_id` parameter overrides the auto-generated `"challenge-{node_id}"` |
| `test_multiple_challenges` | Second challenge on same node gets suffix `-2`; both tracked in `metadata["challenges"]` |
| `test_challenge_nonexistent_raises` | Raises `KeyError` for unknown node IDs |
| `test_dismiss_challenge_restores_target` | `Network.retract("challenge-a")` restores target to `IN` |

### `TestDefend`
Tests `Network.defend(target_id, challenge_id, reason)` — the rebuttal operation.

| Test | Contract Verified |
|------|-------------------|
| `test_defend_restores_target` | Target flips back to `IN`, the challenge node flips to `OUT`; returns `{"defense_id": "defense-challenge-a"}` |
| `test_defense_creates_node` | Creates a node `defense-challenge-a` with `metadata["defense_target"]` and `metadata["defends"]` |
| `test_defend_cascades_restoration` | Restoring target re-activates downstream dependents |
| `test_retract_defense_reinstates_challenge` | Retractions are reversible: retracting the defense re-activates the challenge and re-outs the target |
| `test_defend_nonexistent_target_raises` / `test_defend_nonexistent_challenge_raises` | Both `KeyError` on missing references |

### `TestDialecticalChain`
Tests multi-level argumentation:

- `test_challenge_defense` — Three-step: challenge A → defend A → challenge the defense. Final state: defense `OUT`, original challenge `IN`, A `OUT`.
- `test_three_level_chain` — Four-step: challenge → defense → counter-challenge → counter-defense. Final state: A `IN` (the innermost defense wins).

### `TestDialecticalPersistence`
Single test (`test_round_trip`) that:
1. Builds a challenge + defense on an in-memory `Network`
2. Saves via `Storage(db).save(net)` to a `tmp_path` SQLite file
3. Loads back via `store.load()`
4. Verifies truth values survive
5. Verifies that `retract` on the loaded network still correctly re-activates the challenge

### `TestDialecticalAPI`
Tests the thin `reasons.api` wrappers (which load/save the network around each call):
- `api.challenge(node_id, reason, db_path=...)` → returns same shape as `Network.challenge`
- `api.defend(target, challenge_id, reason, db_path=...)` → returns `{"defense_id": ...}`
- `api.list_nodes(challenged=True, db_path=...)` → filters to only challenged nodes

## Patterns

1. **Arrange-Act-Assert** — Every test builds a minimal network (arrange), calls one dialectical operation (act), then asserts the full resulting state (assert). No shared state between tests; each is self-contained.

2. **Cascade verification** — A recurring pattern is asserting not just the direct effect but the *transitive* effect on dependents (`test_challenge_cascades`, `test_defend_cascades_restoration`). This guards against the common bug of forgetting to propagate truth-value changes.

3. **Reversibility testing** — `retract()` is tested as the inverse of both `challenge` and `defend`. The system treats all dialectical operations as *undoable*, and the tests pin down that the undo semantics are correct at each level.

4. **Layered testing** — The same logical operation is tested at three levels: `Network` (core), `Storage` (persistence), and `api` (public interface). This ensures the API layer is a faithful pass-through and that persistence doesn't corrupt dialectical metadata.

5. **Auto-naming convention** — Challenge IDs follow `challenge-{node_id}` (with `-N` suffix for duplicates); defense IDs follow `defense-{challenge_id}`. Tests assert these exact strings, making the naming contract explicit.

## Dependencies

**This file imports:**
- `pytest` — test framework
- `reasons.Justification` — data class defining antecedent relationships
- `reasons.network.Network` — core graph with `add_node`, `challenge`, `defend`, `retract`
- `reasons.storage.Storage` — SQLite persistence (`save`/`load`)
- `reasons.api` — public function wrappers (`init_db`, `add_node`, `challenge`, `defend`, `get_status`, `list_nodes`)

**Depended upon by:**
- Nothing imports this file (it's a leaf test module). It is discovered and executed by `pytest` via the `tests/` directory.

## Flow

```
Network.challenge(node_id, reason)
  │
  ├─ Validate node exists (else KeyError)
  ├─ Create node "challenge-{id}" with text=reason, truth_value=IN
  ├─ Set node.truth_value = OUT
  ├─ Record bidirectional metadata links
  ├─ Cascade: walk dependents, flip them OUT
  └─ Return {challenge_id, changed: [...]}

Network.defend(target_id, challenge_id, reason)
  │
  ├─ Validate both exist (else KeyError)
  ├─ Create node "defense-{challenge_id}" with metadata links
  ├─ Set challenge node truth_value = OUT (neutralised)
  ├─ Set target truth_value = IN (restored)
  ├─ Cascade: re-activate dependents of target
  └─ Return {defense_id}

Network.retract(node_id)
  │
  └─ Inverts the effect: if retracting a challenge → target back to IN;
      if retracting a defense → challenge back to IN, target back to OUT
```

The API layer (`reasons.api.challenge`, etc.) wraps each call in a `load → operate → save` cycle against the SQLite file at `db_path`.

## Invariants

1. **Truth-value dichotomy**: Every node is either `IN` or `OUT`. Challenging sets `OUT`; defending a challenge sets the challenge `OUT` and the target back to `IN`.

2. **Bidirectional metadata**: A challenge node *always* records `challenge_target`, and the target *always* records the challenge in its `challenges` list. Same pattern for defenses (`defense_target` / `defends`).

3. **Cascade completeness**: Any node whose justification antecedents include an `OUT` node must itself be `OUT`. This is asserted explicitly in `test_challenge_cascades` and `test_defend_cascades_restoration`.

4. **Reversibility**: `retract(challenge_id)` undoes `challenge`; `retract(defense_id)` undoes `defend`. The system must support arbitrary undo at any depth in the chain.

5. **Persistence fidelity**: The SQLite round-trip must preserve all truth values, metadata links, and the ability to perform further operations (retract) on the loaded network.

6. **Naming uniqueness**: Multiple challenges on the same node get sequential suffixes (`challenge-a`, `challenge-a-2`, …). The first always gets the bare suffix.

## Error Handling

- **Missing nodes**: Both `challenge()` and `defend()` raise `KeyError` when the target (or challenge) node ID doesn't exist in the network. Tests use `pytest.raises(KeyError)` to pin this contract. There is no silent no-op or warning — it's a hard failure.
- **No custom exception types**: The dialectical module reuses Python's built-in `KeyError` rather than defining a domain-specific exception. This means callers in the API layer must either catch `KeyError` or let it propagate to the HTTP/CLI boundary.
- **No partial-failure states**: Each operation is atomic within the in-memory graph. There's no transactional rollback tested here (e.g., a challenge that cascades to 50 nodes doesn't partially apply), but the persistence layer's `save()` is called after the full mutation, so a crash mid-operation would simply not be persisted.

## Topics to Explore

- [function] `reasons/network.py:challenge` — The core implementation that creates the challenge node, flips truth values, and cascades; understanding its cascade algorithm is critical for debugging propagation bugs
- [function] `reasons/network.py:retract` — The undo logic that must correctly invert both challenges and defenses; its correctness depends on tracking which operation created each node
- [file] `reasons/api.py` — The public API wrappers that handle the load-operate-save cycle; shows how `db_path` threading works and how errors are surfaced to callers
- [file] `reasons/contradictions.py` — The sibling module for contradiction detection; understanding how challenges differ from (and potentially interact with) contradictions will clarify the full argumentation model
- [general] `doyle-section-6` — The theoretical grounding in Doyle's logic of justification; the challenge/defend pattern maps to his "defeater" concept and understanding the formal semantics will help when extending the system

## Beliefs

- `challenge-creates-persistent-node` — Every call to `Network.challenge()` creates a new first-class node (ID `challenge-{target}` or custom) in the graph, not merely a flag on the target; this node can itself be challenged, defended, or retracted independently.
- `defense-neuters-challenge-via-truth-value` — Defending against a challenge works by flipping the challenge node's truth value to `OUT`, not by deleting it; the challenge node remains in the graph and can be re-activated by retracting the defense.
- `cascade-flips-downstream-dependents` — Challenging a node with `SL`-type justifications causes all transitive dependents to flip to `OUT` in the same operation, and this is reported in the `changed` list of the return value.
- `dialectical-state-survives-sqlite-roundtrip` — After `Storage.save()` → `Storage.load()`, the loaded network retains all challenge/defense nodes with correct truth values and metadata links, and `retract()` on the loaded network correctly re-activates prior challenges.
- `api-layer-is-load-operate-save` — Every `reasons.api` dialectical function (`challenge`, `defend`) loads the network from `db_path`, mutates it in memory, and saves back; there is no long-lived in-process cache between calls.
