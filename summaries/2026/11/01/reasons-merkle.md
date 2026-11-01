---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/merkle.py

# `reasons/merkle.py` — Merkle Tree Integrity for Justification Chains

## Purpose

This module provides **tamper-evident hashing** for a belief/justification network. The core idea: every node in the network has a `text_hash`, and every justification (edge) has a `content_hash` that recursively incorporates all antecedent nodes up the chain. If someone mutates the text of *any* node anywhere in a justification chain, the Merkle root at the top will no longer match the stored hash — giving you a cryptographic proof of integrity.

It owns two responsibilities:
1. **Computation** — deriving the correct hash values from current network state.
2. **Verification & repair** — detecting mismatches and backfilling hashes for nodes/justifications that were created before this system was introduced.

## Key Components

### Hashing Primitives

| Function | Contract |
|----------|----------|
| `compute_text_hash(text) -> str` | Returns `sha256(text.encode("utf-8")).hexdigest()`. Pure, deterministic. |
| `_fresh_merkle_hash(node_id, net, cache, visiting) -> str` | Recursively computes the full Merkle hash for a node **from scratch** (never reads stored hashes). Uses `cache` for memoization and `visiting` for cycle detection. |
| `_fresh_content_hash(text_hash, justification, net, cache, visiting) -> str` | Computes a justification's content hash: `sha256(text_hash \| sorted(antecedent_merkle_hashes))`. |

### Public API

| Function | Contract |
|----------|----------|
| `compute_merkle_hash(node_id, net, cache=None) -> str` | Entry point: compute the full recursive Merkle hash for a single node. Creates a fresh cache if none provided. |
| `compute_content_hash(node, justification, net, cache=None) -> str` | Compute the content hash for one specific justification (used by `backfill_hashes`). |
| `verify_node(node) -> dict \| None` | Compares `node.text_hash` against a fresh `sha256(node.text)`. Returns a finding dict on mismatch. |
| `verify_justification(node, j_index, net, cache=None) -> dict \| None` | Compares `justification.content_hash` against a recomputed value. Returns a finding dict on mismatch. |
| `verify_all(net) -> dict` | Iterates all nodes/justifications, returns `{"findings": [...], "missing_hashes": int}`. |
| `backfill_hashes(net) -> dict` | Populates `text_hash` / `content_hash` on any node/justification that lacks them. Returns counts. |

## Patterns

- **Memoization via mutable cache dict**: Every recursive function accepts a `cache: dict[node_id, str]` that prevents exponential re-computation in shared DAGs. The public wrappers create a fresh `cache={}` so callers don't accidentally share state.

- **Cycle detection via `visiting` set**: If a node appears in its own ancestor chain (a cycle in the justification DAG), `_fresh_merkle_hash` returns `compute_text_hash("")` — the hash of an empty string — rather than recursing infinitely. This is a *silent degradation*, not an error.

- **Sorted antecedents for determinism**: In `_fresh_content_hash`, antecedent hashes are collected with `sorted(...)`. This guarantees that the Merkle root is identical regardless of the order antecedents are stored in the list.

- **"Fresh" computation philosophy**: The `_fresh_*` prefix is intentional — these functions **never** read `node.text_hash` or `j.content_hash` from the network. They always recompute from `node.text`. This is what makes verification sound: you're comparing a freshly-derived value against the stored one, not circularly trusting the stored value.

- **Finding-object error reporting**: `verify_node` and `verify_justification` never raise. They return `None` (clean) or a structured `dict` with a `"type"` discriminator (`"text_mutation"` or `"chain_mutation"`). This lets `verify_all` accumulate findings without try/except.

## Dependencies

- **Imports**: Only `hashlib` (standard library). No third-party dependencies.
- **Imported by**:
  - `reasons/api.py` — likely exposes `verify_all` / `backfill_hashes` as REST endpoints.
  - `reasons/network.py` — the `Network` class probably calls `backfill_hashes` on load or after mutations.
  - `tests/test_merkle.py` — unit/integration tests for hash correctness and cycle handling.

The module is deliberately dependency-free (besides `hashlib`) so it can be unit-tested in isolation and reused without pulling in the full network stack.

## Flow

### Hash Computation (bottom-up recursion)

```
Node C  (text: "Z")
  ↑ justified by J2 (antecedents: [A, B])
Node A  (text: "X")   Node B  (text: "Y")

compute_merkle_hash("C"):
  1. text_hash_C = sha256("Z")
  2. _fresh_content_hash(text_hash_C, J2):
       a. _fresh_merkle_hash("A") → sha256("X")          [leaf, no justifications]
       b. _fresh_merkle_hash("B") → sha256("Y")          [leaf]
       c. content_hash = sha256("sha256(Z) | sha256(X) | sha256(Y)")   [sorted]
  3. result = sha256("sha256(Z) | content_hash")
```

Key detail at step 2a/b: if A or B themselves have justifications, the recursion continues upward. The `cache` dict ensures that if both J1 and J2 reference node A, A's hash is computed once.

### Verification

`verify_all` iterates `sorted(net.nodes.items())` for deterministic output ordering. For each node it checks:
1. `node.text_hash` vs `sha256(node.text)` — a **local** integrity check.
2. Each `justification.content_hash` vs a full recursive recomputation — a **chain** integrity check.

If a hash is `None`/empty, it's counted in `missing_hashes` but **not** treated as a finding. This distinguishes "never hashed" (benign, fixable by `backfill_hashes`) from "hashed but mutated" (a real integrity violation).

### Backfill

`backfill_hashes` walks all nodes, sets `text_hash` where missing, then sets each justification's `content_hash` using the (now-populated) text hashes. It reuses a single `cache` dict across all justifications to avoid redundant recursion.

## Invariants

1. **Determinism**: The same network state always produces the same hash. Sorting antecedents and using UTF-8 encoding guarantee this.
2. **Acyclic assumption with graceful fallback**: The code *tolerates* cycles (returns empty-string hash) but the network is expected to be a DAG. A cycle silently weakens the hash — it won't crash, but the hash no longer reflects the full chain.
3. **`supporting_justification` is the single chain path**: `_fresh_merkle_hash` only follows `node.supporting_justification` (one justification per node), not all justifications. The Merkle chain is a *path*, not a *fan-out*.
4. **Fresh computation ≠ stored value**: The module enforces that verification is always "recompute from text and compare," never "compare stored hash to stored hash."
5. **Cache is scoped per top-level call**: `compute_merkle_hash` and `compute_content_hash` create fresh caches, so callers can't accidentally poison or share memoization across unrelated queries.

## Error Handling

This module **never raises exceptions** under normal operation. Error semantics:

- **Cycle encountered**: Silent. Returns `sha256("")`. The caller has no way to distinguish "cycle" from "node with empty text and no justifications" without knowing the network topology.
- **Antecedent not in `net.nodes`**: Skipped silently (`if a in net.nodes` filter in `_fresh_content_hash`). A dangling reference is treated as "not part of the hash."
- **Hash mismatch**: Reported as a structured finding dict. The caller (likely `api.py` or CLI) decides how to surface it (HTTP 409, CLI warning, etc.).
- **Missing hash**: Tracked in a counter, not a finding. Callers can distinguish "needs backfill" from "actually corrupted."
- **Invalid `j_index`**: `verify_justification` returns `None` (treated as "nothing to verify") rather than raising `IndexError`.

## Topics to Explore

- [file] `reasons/network.py` — Defines the `Network`, `Node`, and `Justification` data structures that this module operates on; understanding `supporting_justification` and `antecedents` semantics is essential.
- [file] `tests/test_merkle.py` — Tests that pin down the expected hash values, cycle behavior, and edge cases (empty antecedents, dangling refs, multi-level chains).
- [function] `reasons/api.py:verify` (or equivalent) — See how the findings from `verify_all` are surfaced to callers (HTTP status codes, CLI output, notification).
- [file] `reasons/verify.py` — A separate verification module; understand the boundary between *integrity verification* (this file) and *logical/semantic verification* (that file).
- [general] `merkle-chain-semantics` — The design choice to hash only the `supporting_justification` path (not all justifications) means the integrity guarantee is per-path, not per-node. Worth understanding the threat model this protects against.

## Beliefs

- `merkle-hash-is-path-scoped` — The Merkle hash for a node incorporates only its `supporting_justification` and that justification's antecedents; other justifications on the same node do not affect the hash.
- `verify-all-never-raises` — `verify_all` returns a structured dict (`findings` + `missing_hashes`) and does not raise exceptions; all integrity issues are data, not control flow.
- `fresh-computation-ignores-stored-hashes` — The `_fresh_*` family of functions computes hashes exclusively from `node.text` and network topology; they never read `node.text_hash` or `j.content_hash`, making verification non-circular.
- `cycle-produces-empty-hash` — A cyclic justification chain does not raise or loop; it silently produces `sha256("")` for the back-edge, weakening (but not breaking) the integrity guarantee.
- `backfill-before-verify-required` — Calling `verify_all` on a network with unhashed nodes produces `missing_hashes > 0` but zero findings for those nodes; `backfill_hashes` must be called first to make verification meaningful.
