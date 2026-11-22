---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: backfill_hashes in reasons/merkle.py

## `backfill_hashes` — Explanation

### 1. Purpose

`backfill_hashes` is a **repair/initialization** routine: it walks every node in a justification network and fills in any missing Merkle hashes with values derived from the *current* node text. It exists so that a network loaded from storage / migration / construction can be hydrated with the integrity hashes required by `verify_all` / `verify_node` / `verify_justification`, without touching hashes that already exist.

It is the "write side" counterpart to the `verify_*` family: **backfill writes; verify checks.** It deliberately does not verify or overwrite existing hashes.

### 2. Contract

**Preconditions (assumed, not enforced):**
- `net` has a `nodes` attribute: a mapping `node_id -> node`.
- Each `node` has:
  - `text: str` — non-`None` (otherwise `compute_text_hash` raises `AttributeError` on `.encode`).
  - `text_hash: Optional[str]` — falsy means "missing" (`None` or `""`).
  - `justifications: list` — each item has `content_hash: Optional[str]` and `antecedents: list[node_id]`.
  - `supporting_justification: Optional[int]` — used by `_fresh_merkle_hash` when a node is used as an antecedent.
- Node IDs referenced in `antecedents` are permitted to be absent (the guard `if a in net.nodes` filters them out); they contribute nothing to the hash.

**Postconditions:**
- Every node with a falsy `text_hash` now holds `sha256(node.text)`.
- Every justification with a falsy `content_hash` now holds the Merkle root computed from current text (`compute_content_hash`).
- Nodes/justifications with a non-empty hash are **left untouched** — even if they are wrong.
- Returned counters reflect exactly how many fields were assigned during this call.

**Invariants:**
- No node text is mutated.
- No existing (truthy) hash is recomputed or overwritten.
- The `cache` stores Merkle hashes derived from current text; it is consistent for the duration of the call because text does not change.
- Existing hashes are *not* validated — a "backfilled" network can still contain stale/incorrect hashes and should be followed by `verify_all` if validation is desired.

### 3. Parameters

- **`net`** — the network object. Expected to expose `net.nodes` as a dict-like mapping. Also each `node` must be compatible with `compute_text_hash` / `compute_content_hash`. Edge case: a `node.text` of `None` will raise; an empty `node.text` is fine (`sha256(b"")`).

There are no other parameters — `cache` is created internally and discarded.

### 4. Return Value

```python
{
    "nodes_updated": int,        # count of node.text_hash assignments
    "justifications_updated": int,  # count of justification.content_hash assignments
}
```

The caller gets *counts only* — not the list of updated ids, not the cache, not a diff. If no changes were needed, both counts are `0`. There's no error signal; callers must independently verify results (e.g., `verify_all(net)`) if they need to know whether the resulting hashes are correct.

### 5. Algorithm

1. Initialize `nodes_updated = 0`, `justifications_updated = 0`, and a shared `cache = {}`.
2. Iterate `net.nodes.items()` **in dict order** (insertion order in modern CPython).
3. For each `node`:
   - If `node.text_hash` is falsy: set `node.text_hash = compute_text_hash(node.text)` and increment `nodes_updated`.
   - For each `j in node.justifications`:
     - If `j.content_hash` is falsy: set `j.content_hash = compute_content_hash(node, j, net, cache)` and increment `justifications_updated`.
4. Return the counters.

Key subtleties of the delegated computation:

- `compute_content_hash(node, j, net, cache)` **recomputes** `text_hash = compute_text_hash(node.text)` internally — it does **not** read the `node.text_hash` that step 3 may have just set. So the content hash is always derived from live text, never from stored hashes.
- The content hash is `sha256(node_text_hash + "|" + "|".join(sorted(antecedent_merkle_hashes)))`, matching `_fresh_content_hash`. For each antecedent `a`, `_fresh_merkle_hash(a, …)` is called; if that antecedent has a `supporting_justification`, its own Merkle hash chains through that justification's antecedents. This is how the tree/chains propagate transitively.
- The shared `cache` memoizes `_fresh_merkle_hash` results by `node_id`, so multiple justifications referencing the same antecedents compute each antecedent's Merkle hash at most once per distinct `visiting` path. **Note:** the cache is keyed only by `node_id`, not by which justification is being computed — valid here because only `node.text` (immutable across this function) feeds Merkle hashes.
- Cycle safety: `_fresh_merkle_hash` uses a `visiting` set. A node already on the current recursion path yields `compute_text_hash("")`, preventing infinite recursion. Cycles are thus hashed deterministically (to a degenerate value) rather than crashing.

Iteration order does not affect the computed content hashes because all Merkle hashes derive from live text, not from the stored `text_hash`/`content_hash` fields being filled.

### 6. Side Effects

- **Mutates in place**: `node.text_hash` and `j.content_hash` on the objects reachable from `net.nodes`.
- **Local cache** populated but discarded on return — not observable to callers.
- **No I/O**, no logging, no exceptions raised for the normal path, no external calls.
- Does not touch `net` beyond reading `net.nodes`.

### 7. Error Handling

There is **no explicit error handling** — errors propagate:

- `AttributeError` if `node.text` is not a `str` (e.g., `None`) because `compute_text_hash` calls `text.encode("utf-8")`.
- `AttributeError`/`TypeError` if `node.justifications` is not iterable, or `justification.antecedents` is not iterable.
- `KeyError` is **mostly avoided**: `_fresh_content_hash` filters antecedents via `if a in net.nodes`. However, `_fresh_merkle_hash` (reached via `compute_merkle_hash`, *not* in this path) has no such guard — in `backfill_hashes` the only accessible entry to `_fresh_merkle_hash` is through `_fresh_content_hash`, which is guarded.
- A `RecursionError` is prevented by the `visiting` set.
- Silent "successful" partial failure is not possible: any exception aborts the loop, leaving earlier nodes/justifications mutated and later ones untouched. Callers should treat an exception as "state partially backfilled" and re-run idempotently (which is safe, since falsy-only checks make it idempotent).

### 8. Usage Patterns

Typical callers:

- **Bootstrap / migration**: after loading nodes that predate hashing support, call `backfill_hashes(net)` once, then `verify_all(net)` to catch tampering (backfill can't detect it; verify can).
- **Post-import hydration**: after constructing nodes via an importer that doesn't populate hashes.
- **Idempotent repair**: safe to call repeatedly; the second call returns `{0, 0}`.

Caller obligations:
- Ensure `node.text` is a `str` before calling (a common source of `AttributeError`).
- Do not expect backfill to *fix* incorrect hashes — only to fill *missing* ones.
- Persist the mutated `net` afterwards if durability is required (backfill only mutates in memory).

### 9. Dependencies

Internal (same module):
- `compute_text_hash` — `sha256(text).hexdigest()`.
- `compute_content_hash` — recomputes text hash then calls `_fresh_content_hash`.
- `_fresh_content_hash` and `_fresh_merkle_hash` — the recursive Merkle construction with `cache`/`visiting` handling.

External:
- `hashlib` (standard library) — for SHA-256.

Transitively, it depends on `node.justifications`, `justification.antecedents`, and `node.supporting_justification` being shaped as the rest of `merkle.py` expects.

### Assumptions not enforced by the type system

- `node.text_hash` and `j.content_hash` are "missing" iff falsy (`None` or `""`). A value of `0` or `False` would also be treated as missing — harmless here but worth knowing if schemas evolve.
- `node.text` is a `str` (not `bytes`, not `None`).
- `supporting_justification`, when set, is a valid index into `node.justifications`; a dangling index raises `IndexError` inside `_fresh_merkle_hash` when that node is an antecedent.
- Referenced antecedent IDs that are absent from `net.nodes` are silently skipped — the hash is computed *as if* they didn't exist, which is not the same as an error but can yield a misleadingly stable-looking hash.
- Node text is treated as immutable during the call; mutating `node.text` between iterations would invalidate the shared cache. Nothing enforces this.

## Topics to Explore

- [file] `tests/test_merkle.py` — Confirms expected counts, idempotency, tamper detection, and cycle behavior for `backfill_hashes`.
- [function] `reasons/merkle.py:compute_content_hash` — The exact hash formula backfill writes into `justification.content_hash`.
- [function] `reasons/merkle.py:_fresh_merkle_hash` — Explains how transitive/antecedent chaining and cycle handling work.
- [function] `reasons/merkle.py:verify_all` — The natural post-backfill validation pass; documents the `findings`/`missing_hashes` contract.
- [general] `backfill-callers` — Search the repo for other callers of `backfill_hashes` to see whether text mutation, persistence, or verification is performed around it.

## Beliefs

- `backfill-only-fills-falsy-hashes` — The function assigns `text_hash`/`content_hash` only when the existing value is falsy, so it never overwrites or validates existing hashes.
- `backfill-computes-from-current-text` — Content hashes are computed from live `node.text`, not from any `text_hash` written earlier in the same call, because `compute_content_hash` recomputes it internally.
- `backfill-is-idempotent` — A second call on the same network returns `{"nodes_updated": 0, "justifications_updated": 0}` as long as no hash fields were cleared.
- `backfill-mutates-in-place` — It mutates `node.text_hash` and `justification.content_hash` on objects reachable from `net.nodes`; the internal `cache` is discarded.
- `backfill-skips-missing-antecedents` — Antecedent IDs not present in `net.nodes` are excluded from the content hash via `if a in net.nodes`, silently producing a hash that omits them.
