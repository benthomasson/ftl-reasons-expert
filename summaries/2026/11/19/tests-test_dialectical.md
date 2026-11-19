---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: tests/test_dialectical.py

# `tests/test_dialectical.py` — Explanation

## Purpose

This file is the acceptance suite for the **dialectical argumentation** feature of the `reasons` library. It verifies that a network of beliefs can be subjected to a multi-round challenge/defend cycle — conceptually borrowed from Doyle's truth-maintenance systems where arguments themselves become first-class, challengeable nodes. The suite exercises the feature at three layers:

1. **In-memory `Network`** (`TestChallenge`, `TestDefend`, `TestDialecticalChain`)
2. **Persistence** via `Storage` on SQLite (`TestDialecticalPersistence`)
3. **Public API** functions (`api.challenge`, `api.defend`, `api.list_nodes`, …) backed by a real DB (`TestDialecticalAPI`)

Because a reasoning engine's *behavior* under contradiction is what makes or breaks the product, this file is effectively the executable spec for "what happens when you challenge a belief and how do you retract a defense."

---

## Key Components

### Test classes (grouped by contract)

| Class | Contract under test |
|---|---|
| `TestChallenge` | `Network.challenge(target, reason, challenge_id=None)` flips `target.truth_value` to `"OUT"`, creates a `challenge-<id>` node, cascades to dependents, and records metadata. |
| `TestDefend` | `Network.defend(target, challenge_id, reason)` neutralizes the challenge (puts `challenge_id` OUT) and restores `target` to `"IN"`, cascading restoration. |
| `TestDialecticalChain` | Multi-level chains: you can challenge a defense, then defend *that* challenge, and truth values resolve correctly across the whole graph. |
| `TestDialecticalPersistence` | A challenge+defend sequence survives a `Storage.save` → `Storage.load` round-trip; retraction logic still works after reload. |
| `TestDialecticalAPI` | The functional API using `db_path=` persists the same behavior at the CLI/server boundary. |

### Contracts exposed in the tests

- **`Network.challenge(target, reason, challenge_id=None) -> dict`**
  - Returns `{"challenge_id": "...", "changed": [...]}`.
  - Default ID is `challenge-<target>`; on collision, `challenge-<target>-2`, `-3`, … (see `test_multiple_challenges`).
  - After call: `target.truth_value == "OUT"`, a new node `challenge-<target>` exists with `truth_value="IN"`, `text=reason`, `metadata["challenge_target"] == target`; and `target.metadata["challenges"]` contains the new ID.
  - Raises `KeyError` if `target` does not exist.

- **`Network.defend(target, challenge_id, reason) -> dict`**
  - Returns `{"defense_id": "...", ...}`.
  - Default defense ID = `defense-<challenge_id>` (e.g. `defense-challenge-a`).
  - After call: `challenge_id.truth_value == "OUT"`, `target.truth_value == "IN"`, new defense node has `metadata["defense_target"] == challenge_id` and `metadata["defends"] == target`.
  - Raises `KeyError` if either `target` or `challenge_id` is missing.

- **`Network.retract(node_id)`** — generic retraction used to "undo" a challenge or defense:
  - Retract a challenge → target returns to `IN`.
  - Retract a defense → challenge returns to `IN`, and target goes `OUT` again.

- **`api.challenge`, `api.defend`, `api.get_status`, `api.list_nodes(challenged=True)`** — same semantics, but persist to `db_path`.

### Fixture usage

- `tmp_path` (pytest built-in) is used for every DB-backed test to guarantee isolation — no shared SQLite file leaks across tests.

---

## Patterns

- **Arrange / Act / Assert (AAA)** in every test — no shared mutable state, each test builds a fresh `Network()`.
- **Layered testing**: the same logical operation (challenge → defend → retract) is asserted at the in-memory layer, the storage layer, and the API layer. This is a deliberate "contract-first" progression — if the network is correct but the API isn't, only `TestDialecticalAPI` fails.
- **Dialectical / argumentation-framework model**: challenges and defenses are *nodes in the same graph* rather than side-band flags. This is what lets `TestDialecticalChain` challenge a defense — the defense itself is a first-class assertion.
- **Metadata-based graph edges**: instead of a dedicated edge table, edges are encoded as:
  - `challenge_node.metadata["challenge_target"] = target`
  - `target.metadata["challenges"] += [challenge_id]`
  - `defense_node.metadata["defense_target"] = challenge_id`
  - `defense_node.metadata["defends"] = target`
- **Naming convention as protocol**: `challenge-*` / `defense-*` prefixes are load-bearing — `retract` and cascade logic key off node structure, and the chained test literally depends on `defense-challenge-a` being how a defense of `challenge-a` is identified when you want to *challenge* it.
- **Automatic truth-value propagation**: no test manually re-evaluates; each `challenge`, `defend`, and `retract` is expected to recompute downstream dependents (see `test_challenge_cascades` and `test_defend_cascades_restoration`). This is the TMS "well-founded model" recomputation.

---

## Dependencies

### Imported by this file
- `pytest` — test runner + `tmp_path` + `pytest.raises`.
- `reasons.Justification` — constructs `Justification(type="SL", antecedents=[...])` to establish derivation links.
- `reasons.network.Network` — the in-memory reasoning graph (`.nodes`, `.add_node`, `.challenge`, `.defend`, `.retract`).
- `reasons.storage.Storage` — SQLite persistence (`.save(net)`, `.load()`, `.close()`).
- `reasons.api` — functional DB-backed API (`init_db`, `add_node`, `challenge`, `defend`, `get_status`, `list_nodes`).

### Imported by
- **Nothing directly** — this is a leaf test module. It is discovered by `pytest` (and probably the GitHub Actions workflow, since `.github/workflows` exists in the repo).

### Implicit dependencies to keep in mind when maintaining it
- The `reasons_lib` directory mirrors `reasons` and lacks `merkle.py`/`check_stale.py`; the tests target `reasons`, so any refactor that touches `reasons/network.py` or `reasons/api.py` must keep this contract intact.
- Node ID conventions (`challenge-a`, `defense-challenge-a`) are hardcoded in tests — renaming them breaks the suite, but also requires a matching change in the chain test's `defend("defense-challenge-a", "challenge-defense-challenge-a", ...)` expectation.

---

## Flow

Per test, the canonical execution path is:

```
add_node(a)                    → a.truth_value = "IN"
challenge(a, "wrong")
    ├─ create node "challenge-a" (IN)
    ├─ link metadata both directions
    ├─ set a.truth_value = "OUT"
    └─ propagate dependents → OUT   (changed list includes downstream)
defend(a, "challenge-a", "right")
    ├─ create node "defense-challenge-a" (IN)
    ├─ set challenge-a.truth_value = "OUT"
    └─ a.truth_value = "IN" → cascade dependents back to IN
retract("defense-challenge-a")
    ├─ defense node disappears / becomes non-supporting
    └─ challenge-a returns to IN → a returns to OUT
```

The three-level chain shows this is **recursive**:

```
challenge(a)                  → a OUT
defend(a, challenge-a)        → challenge-a OUT, a IN
challenge(defense-challenge-a)
                              → defense-challenge-a OUT, challenge-a IN, a OUT
defend(defense-challenge-a,
       challenge-defense-challenge-a)
                              → defense-defense IN, challenge-defense OUT,
                                defense-challenge-a IN, challenge-a OUT, a IN
```

The key transformation is `truth_value` recomputation up the *justification* graph (derived nodes) and along the *challenge/defense* graph (dialectical nodes). `Storage.save`/`load` must preserve both graphs plus metadata so the reloaded object recomputes identically (`TestDialecticalPersistence`).

---

## Invariants

These are the rules the tests enforce — treat them as the module's public spec:

1. **Freshly added nodes are `IN`** (`test_challenge_premise`).
2. **A challenge always flips `target.truth_value` to `OUT`** and adds the challenge ID to `target.metadata["challenges"]`.
3. **Cascading is automatic** on every truth-value mutation — both directions (going OUT and coming back IN).
4. **`challenge-<target>` is the default ID; collisions append `-2`, `-3`, …** (`test_multiple_challenges`).
5. **`defense-<challenge_id>` is the default defense ID** (`result["defense_id"] == "defense-challenge-a"`).
6. **A defense puts the challenge it targets `OUT`** — two such defenses would be a contradiction the tests do not cover, but the invariant is one-defense-per-challenge in the default naming scheme.
7. **`retract` reverses whatever that node was doing** — it is the dialectical "undo" — and correctly re-propagates.
8. **The `changed` list returned by `challenge` includes both the target and every cascaded dependent** (see `test_challenge_cascades` asserting `"b" in result["changed"]`).
9. **Persistence is behavior-preserving**: after `Storage.load()`, both `truth_value`s and metadata are identical, and subsequent `retract` produces the same downstream effect.
10. **`list_nodes(challenged=True)` filters to nodes that have been challenged** — in `test_list_challenged` only `a` qualifies (the `challenge-a` node itself is excluded from the result, and `b` — an unchallenged premise — is filtered out).

---

## Error Handling

- **KeyError on unknown `target`/`challenge_id`**: `test_challenge_nonexistent_raises`, `test_defend_nonexistent_target_raises`, and `test_defend_nonexistent_challenge_raises` all pin down that missing graph references raise `KeyError` rather than silently creating stubs. This is important because the rest of the library can then rely on `try/except KeyError` to mean "not found".
- **No swallowing in the test layer** — the tests deliberately do not wrap calls in `try` except where asserting the raise. This ensures the API surface's failure modes are unambiguous.
- **Calling `defend` on an existing target but a missing challenge** is checked separately from a missing target (`test_defend_nonexistent_challenge_raises`), so implementations must validate both arguments, not just the target.
- **No explicit error tests at the API layer** — error semantics for `api.challenge`/`api.defend` with bad IDs are not asserted here, so the behavior for those cases should be checked in `tests/test_api.py`.

---

## Topics to Explore

- [file] `reasons/network.py` — Source of `Network.challenge`, `Network.defend`, and the cascade re-evaluation logic that this suite pins down.
- [file] `reasons/api.py` — The DB-backed `api.challenge`, `api.defend`, `api.get_status`, and `api.list_nodes(challenged=True)` implementations that `TestDialecticalAPI` verifies.
- [file] `reasons/storage.py` — How `Storage.save`/`Storage.load` serializes challenge/defense nodes and metadata so `TestDialecticalPersistence` round-trips correctly.
- [general] `doyle-section-6-tms` — The paper/model referenced in the module docstring; understanding it clarifies why challenges and defenses are modeled as nodes rather than flags.
- [file] `tests/test_defeat_justification.py` — Likely the sibling of this file for the *defeat* (rather than *dialectical*) mode; useful for comparing how two different conflict-resolution strategies are implemented over the same graph.

## Beliefs

- `challenge-returns-dict-with-challenge-id-and-changed` — `Network.challenge` is contractually required to return `{"challenge_id": ..., "changed": [...]}`, and `"changed"` includes every node whose truth value was affected, including cascaded dependents.
- `defense-id-is-derived-from-challenge-id` — The default defense node ID is `defense-<challenge_id>`, making the defense's identity a deterministic function of the challenge it neutralizes.
- `challenge-id-collisions-get-numeric-suffix` — A second call to `challenge(target)` with the same target produces `challenge-<target>-2`, not an error and not an overwrite.
- `retract-is-the-symmetric-undo-for-challenge-and-defense` — Retracting a challenge restores the target to `IN`; retracting a defense reinstates the challenge and puts the target back to `OUT` — the same `Network.retract` primitive is used for both.
- `dialectical-nodes-are-metadata-linked-not-edge-tabled` — Challenges and defenses attach to the graph via `metadata["challenge_target"]` / `metadata["challenges"]` / `metadata["defense_target"]` / `metadata["defends"]`, not via a separate edge collection.
- `missing-graph-references-raise-keyerror` — `challenge` and `defend` both raise `KeyError` (not a custom exception, not a silent no-op) when the target or challenge node is absent.
