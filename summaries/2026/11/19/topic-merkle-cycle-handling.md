---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# Topic: Evaluate whether silently converting cycles to `sha256("")` is desired, or whether cycles should be reported as structural findings.

## Evaluation

**No — silently converting a cycle to `sha256("")` is not a good final behavior for an integrity-verification path.** It is reasonable as a defensive recursion guard to prevent infinite loops, but the current implementation makes the cycle invisible and can let a structurally invalid graph verify as clean.

The behavior is here:

- `reasons/merkle.py:17-22` — `_fresh_merkle_hash` checks `if node_id in visiting: return compute_text_hash("")`.  
  `compute_text_hash("")` is effectively `sha256("")`. The earlier grep missed this because the code calls `compute_text_hash("")`, not literal `sha256("")`.

Why this is problematic:

1. **It can produce false-clean verification.**  
   `verify_all` (`reasons/merkle.py:110-127`) only compares stored hashes to recomputed hashes. If the stored hash was created or backfilled by the same cycle-tolerant code, a cyclic graph can recompute to the same masked value and produce no finding.

2. **`backfill_hashes` can silently bless cycles.**  
   `backfill_hashes` (`reasons/merkle.py:130-146`) calls `compute_content_hash` for every missing justification hash. Because `compute_content_hash` ultimately uses the same cycle guard, a cyclic justification can receive a normal-looking hash instead of being reported as invalid.

3. **The API has no structural cycle finding.**  
   `reasons/api.py:2736-2740` imports `verify_all` and partitions findings into `text_mutations` and `chain_mutations`. There is no `cycle` finding type. If a cycle is introduced after hashes are written, it may show up as a generic `chain_mutation`, but it will not identify the structural problem.

4. **The hash is ambiguous and traversal-dependent.**  
   `sha256("")` is a legitimate text hash for an empty string, so it is a bad sentinel. More importantly, the cycle cut depends on the current `visiting` set and cache context. A cyclic component can therefore have different “Merkle roots” depending on where recursion starts, which undermines the integrity model.

5. **The test appears to check safety, not correctness.**  
   `tests/test_merkle.py:193-200` defines `test_cycle_safety`, whose visible docstring says it verifies the code “doesn't infinite-loop.” The observations truncate at line 200, so the actual assertions are missing. If the test expects a clean result rather than a structural finding, that codifies avoidance rather than detection.

**Recommendation:** keep cycle protection, but make cycles explicit structural findings. For example, `_fresh_merkle_hash` could raise a `CycleError` when `node_id in visiting`, and `verify_all` could catch that and emit `{"type": "cycle", ...}`. `backfill_hashes` should refuse to write hashes for cyclic components rather than normalizing them. If exceptions are undesirable, return a distinct result object or sentinel that verification always reports. If the public API is supposed to prevent cycles, the guard is for corrupted data and should still be loud.

The observations are insufficient to know whether normal node creation/update code prevents cycles. That requires inspecting `reasons/network.py` and the rest of `tests/test_merkle.py`.

## Topics to Explore

- [file] `tests/test_merkle.py` — The remainder of `test_cycle_safety` after line 200 is needed to see whether it only checks termination or asserts a clean/no-finding result.
- [file] `reasons/network.py` — Inspect where `compute_content_hash` and `compute_text_hash` are called and whether normal creation/update paths reject cycles before hashing.
- [function] `reasons/api.py:check_integrity` — This function partitions Merkle findings into `text_mutations` and `chain_mutations`; a new `cycle` finding needs plumbing here and in CLI output.
- [function] `reasons/merkle.py:verify_all` — This is the natural place to catch and report cycle findings instead of allowing `_fresh_merkle_hash` to hide them.
- [general] `cycle-detection-contract` — Decide whether cycle detection should raise, return a sentinel, or emit structural findings consistently across verification and backfill paths.

## Beliefs

- `merkle-cycle-guard-returns-empty-text-hash` — `_fresh_merkle_hash` returns `compute_text_hash("")` when `node_id` is already in `visiting` (`reasons/merkle.py:21-22`).
- `verify-all-has-no-cycle-finding-type` — `verify_all` and its helpers only produce `text_mutation` and `chain_mutation` finding dicts; no cycle finding type exists (`reasons/merkle.py:70-127`, `reasons/api.py:2739-2740`).
- `backfill-hashes-can-silently-hash-cycles` — `backfill_hashes` calls `compute_content_hash` for every missing justification hash, so cyclic justifications can be assigned normal-looking hashes (`reasons/merkle.py:130-146`).
- `check-integrity-maps-findings-to-two-keys` — `api.check_integrity` exposes only `text_mutations` and `chain_mutations`, so a new structural cycle finding would require API/CLI changes (`reasons/api.py:2736-2740`).
- `cyclic-merkle-hash-depends-on-traversal` — Because the back-edge is replaced with a constant only while a node is in the current `visiting` set, the resulting hash for a cyclic component depends on traversal/cache order (`reasons/merkle.py:17-47`, `reasons/merkle.py:110-127`).
