---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: How `text_hash` and `content_hash` chain across a node's justifications transitively commits to antecedent hashes, which is what makes `chain_mutation` detection possible.

## How the hashes chain

### 1. `text_hash` is a per-node commitment to `node.text`

`text_hash` is just SHA-256 over the node’s text:

- `compute_text_hash()` in `reasons/merkle.py:12-14`
- stored on `Node.text_hash` in `reasons/__init__.py:42`
- persisted in the `nodes` table, and exported/imported via `reasons/api.py:897` and related paths

Verification is simple: `verify_node()` recomputes `sha256(node.text)` and compares it to the stored `node.text_hash` (`reasons/merkle.py:70-85`). A mismatch produces a `text_mutation` finding.

### 2. `content_hash` is a commitment to a justification’s antecedents

A `Justification` stores `content_hash` (`reasons/__init__.py:23`, persisted in `reasons/storage.py:43`). It is computed by `compute_content_hash()` (`reasons/merkle.py:59-67`) through `_fresh_content_hash()` (`reasons/merkle.py:41-49`).

For a node `N` and one of its justifications `j`, the effective formula is:

```text
antecedent_hashes = sorted(
    fresh_merkle_hash(a)
    for a in j.antecedents
    if a in net.nodes
)

content_hash(N, j) =
    sha256(
        fresh_text_hash(N)
        + "|"
        + "|".join(antecedent_hashes)
    )
```

If `j.antecedents` is empty, the content hash is just:

```text
sha256(fresh_text_hash(N))
```

Important details:

- `compute_content_hash()` computes `text_hash = compute_text_hash(node.text)` freshly at `reasons/merkle.py:66`. It does **not** trust the stored `node.text_hash`.
- Antecedent hashes are sorted at `reasons/merkle.py:43-47`, so antecedent order does not affect the hash.
- The recursive antecedent hash is `_fresh_merkle_hash()` (`reasons/merkle.py:17-38`), not the stored `content_hash`.

### 3. A node’s recursive Merkle hash folds in its supporting justification

`_fresh_merkle_hash()` computes a node’s Merkle hash from current text:

```text
fresh_text_hash = sha256(node.text)

if node has no justifications or no supporting_justification:
    merkle_hash(node) = fresh_text_hash
else:
    j = node.justifications[node.supporting_justification]
    ch = content_hash(node, j)
    merkle_hash(node) = sha256(fresh_text_hash + "|" + ch)
```

This is implemented at `reasons/merkle.py:26-35`.

So the recursive structure is:

```text
content_hash(N, justification)
  -> includes fresh_merkle_hash(each antecedent)
       -> includes antecedent text hash
       -> includes antecedent's supporting justification content_hash
            -> includes that antecedent's antecedents recursively
```

That is the transitive commitment: a justification’s `content_hash` commits not only to its direct antecedents’ text, but also to the supporting justification chains of those antecedents.

One nuance: for transitive propagation through a node, only the node’s `supporting_justification` is folded into its Merkle hash (`reasons/merkle.py:28-32`). Other justifications on that node are still individually verified, but they are not part of the node’s Merkle hash used by downstream nodes.

### 4. `chain_mutation` detection compares stored vs freshly recomputed `content_hash`

`verify_justification()` does the actual chain check:

- skip if `j.content_hash` is missing (`reasons/merkle.py:96-97`)
- recompute from the current network state via `compute_content_hash(node, j, net, cache)` (`reasons/merkle.py:98`)
- compare with the stored `j.content_hash`
- if different, emit:

```python
{
    "type": "chain_mutation",
    "node_id": node.id,
    "justification_index": j_index,
    "stored_hash": j.content_hash,
    "current_hash": current,
}
```

That is `reasons/merkle.py:88-107`.

`verify_all()` walks every node and every justification, collects findings, and counts missing hashes (`reasons/merkle.py:110-129`). The API then separates findings into `text_mutations` and `chain_mutations` in `reasons/api.py:2734-2743`, and the CLI reports them in `reasons/cli.py:1165-1167`.

### 5. Why this detects an ancestor mutation even if the ancestor’s `text_hash` is updated

Suppose the chain is:

```text
A cites B, B cites C
```

An attacker mutates `C.text` and also updates `C.text_hash` so that `C` itself looks clean:

- `verify_node(C)` passes because `C.text_hash == sha256(C.text)`.
- But `fresh_merkle_hash(C)` changes because it hashes the current `C.text` at `reasons/merkle.py:26`.
- Therefore `B`’s recomputed justification content hash changes, because it includes `fresh_merkle_hash(C)`.
- `B.justifications[...].content_hash` still stores the old value, so `verify_justification(B, ...)` emits `chain_mutation`.
- `A`’s recomputed content hash includes `fresh_merkle_hash(B)`, which itself includes `B`’s supporting justification content hash, which depends on `C`. So `A` is also flagged.

This is exactly what the tests demonstrate:

- `test_verify_chain_mutation` mutates `p1.text` and updates `p1.text_hash`, then expects zero `text_mutations` but one `chain_mutation` on `d1` (`tests/test_merkle.py:118-139`).
- `test_merkle_transitive` mutates `c.text`, updates `c.text_hash`, and expects both `b` and `a` to be flagged as `chain_mutation` (`tests/test_merkle.py:test_merkle_transitive`).

### 6. Missing hashes and backfill

`chain_mutation` detection requires the stored `content_hash` to exist. If `j.content_hash` is empty, `verify_justification()` returns `None` and does not compare anything (`reasons/merkle.py:96-97`). `verify_all()` counts those as `missing_hashes`.

`backfill_hashes()` fills missing `node.text_hash` and `justification.content_hash` values from the current network state (`reasons/merkle.py:132-145`). So integrity checking is only meaningful for hashes that were present before mutation.

## Topics to Explore

- [file] `reasons/merkle.py` — Contains the full recursive hash construction and verification logic.
- [function] `reasons/merkle.py:_fresh_merkle_hash` — Shows exactly how a node’s supporting justification is folded into its Merkle hash.
- [function] `reasons/merkle.py:verify_justification` — The exact comparison that emits `chain_mutation` findings.
- [function] `reasons/api.py:check_integrity` — Maps verification findings into `text_mutations` and `chain_mutations`.
- [file] `tests/test_merkle.py` — Demonstrates transitive detection, updated-`text_hash` behavior, and backfill behavior.

## Beliefs

- `text-hash-is-sha256-of-node-text` — Each node’s stored `text_hash` is the SHA-256 hex digest of its current `text`, and `verify_node` reports a `text_mutation` when they differ.
- `content-hash-commits-to-sorted-antecedent-merkle-hashes` — A justification’s `content_hash` is SHA-256 over the node’s fresh text hash and the sorted recursive Merkle hashes of its antecedents.
- `node-merkle-hash-requires-supporting-justification` — A node’s recursive Merkle hash includes its supporting justification’s content hash only when `supporting_justification` is set; otherwise it is just the node text hash.
- `chain-mutation-detection-requires-stored-content-hash` — `verify_justification` skips comparison and emits no `chain_mutation` when the justification’s `content_hash` is empty.
- `stored-text-hash-update-does-not-prevent-chain-mutation` — Updating a mutated ancestor’s `text_hash` to match its new text avoids a `text_mutation` finding but still causes descendant `chain_mutation` findings because recomputation uses current text.
