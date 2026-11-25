---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Because `_sync_claims` mutates the in-memory `Network` in place with no try/except, any raised exception leaves the caller's object dirty; explore whether callers reload from DB or reuse the object.

# `_sync_claims` Mutation Safety — Reload vs. Reuse

## What the observations establish

### The function and its call sites

- **Definition:** `reasons/import_agent.py:336`
  ```python
  def _sync_claims(network, agent_name, claims, source_path, nogoods):
  ```
- **Call sites:** `reasons/import_agent.py:582` and `reasons/import_agent.py:599`, both as the tail expression of an enclosing helper:
  ```python
  return _sync_claims(network, agent_name, claims, source_path, nogoods)
  ```

Two things are immediately implied by the call shape:

1. `network` is passed positionally as the first argument — a mutation-in-place contract (not `network`) → new Network.
2. Both sites are `return` expressions. If recovery were local, we would expect a `try:` block at this level; the observations show none — meaning any recovery (if it exists) lives *inside* `_sync_claims` or *above* the enclosing helper.

### The `Network` object is not transactional

From the fully-read `reasons/network.py` (~1,000 lines), `Network` is a plain in-memory graph with mutable containers built in `__init__` (`nodes`, `nogoods`, `repos`, `log`, `meta`). Every mutation method commits eagerly:

- `add_node` — `self.nodes[id] = node` is executed *before* Merkle hashing, `_inherit_access_tags`, and `_compute_truth`, and before `self._log("add", …)`. Any failure downstream leaves a half-initialized node registered, plus `dependents` entries added to antecedent/outlist nodes.
- `retract` — mutates `truth_value`, `metadata["_retracted"]`, `retracted_at`, `updated_at` *before* calling `_propagate`, which in turn mutates dependents' `truth_value` in BFS order and appends audit entries to `self.log`.
- `add_nogood` — `self.nogoods.append(nogood)` and `_log("nogood", …)` happen *before* the culprit search and `retract(victim_id)`.
- `add_justification`, `challenge`, `supersede`, `convert_to_premise`, `remove_justification` — all mutate the same way with no compensating cleanup path.

There is **no** `snapshot`, `restore`, `copy`, `transaction`, `commit`, or `rollback` method anywhere in `Network`. So an exception raised part-way through `_sync_claims` (e.g., midway through a loop over `claims`) leaves the caller's object dirty in exactly the way described: some nodes added, some `dependents` sets extended, some `nogoods` appended, some `truth_value` changes propagated, all logged in `self.log` as if they were intended.

A concrete asymmetry worth flagging: `add_node` raises `ValueError("Node … already exists")` *before* insertion, but any *later* step inside `add_node` (hash computation, truth computation, access-tag inheritance) runs after insertion. So even a single failed `add_node` can leave a phantom node.

## The actual question: do callers reload from DB or reuse the object?

**The observations are insufficient to answer this.** The pieces that would settle it are exactly the pieces missing:

| Missing observation | Why it matters |
|---|---|
| Body of `_sync_claims` (lines 336–~581) | The function may do a pre-flight validation pass, wrap itself in `try/except`, or rebuild a fresh `Network` on error. Without this we cannot tell whether the "dirty object" risk is *mitigated internally*. |
| Enclosing functions around lines 582 and 599 | These are the only callers. We need to see whether they catch exceptions, discard the `network`, or reuse it on the next iteration. |
| Any persistence layer for `Network` | `reasons/network.py` exposes **no** `save`/`load`/`to_dict`/`from_dict`. `pyproject.toml` declares an optional `pg = ["psycopg[binary]>=3.1"]` extra, so persistence likely exists in a module not included in the observations — the caller can only "reload from DB" if such a module exists. |
| Top-level entry point (`reasons/cli.py`, or wherever `import_agent` is invoked) | Recovery granularity may be at the whole-import level, in which case the dirty `Network` is simply discarded when the job aborts and a fresh one is constructed — a coarse reload, not a DB reload. |

### What the two possible patterns would look like here

If callers **reload from DB**:
```python
try:
    _sync_claims(network, agent_name, claims, source_path, nogoods)
except Exception:
    network = load_network_from_store(...)   # fresh in-memory object
    raise
```
or the enclosing helper takes a DB handle and reconstructs the `Network` on retry.

If callers **reuse the object**:
- The `return _sync_claims(...)` at 582/599 would be trusted with no local recovery, and either
  - (safe) `_sync_claims` is itself pre-validated so it cannot raise after mutating, or
  - (unsafe) the caller continues using a partially-mutated `Network` after an exception.

Given that both lines are bare `return` expressions with no `try`, the safest reading is: the enclosing helper does **not** perform local recovery. Any recovery is either inside `_sync_claims` or at a higher level (CLI / job runner), where the `Network` is likely thrown away rather than reused — but this observation cannot confirm that.

### Bottom line

- The mutation risk is **structurally real**: `Network` has no rollback primitive, `add_node` commits before completing, and `_sync_claims` receives the object positionally.
- Whether the risk is **realized** depends entirely on code not present in the observations (the `_sync_claims` body and its callers).
- To resolve "reload vs. reuse," read `reasons/import_agent.py` lines 336–600, then trace where the enclosing helpers are invoked (likely `reasons/cli.py` or an import orchestration module), and finally locate any `Network` persistence module (search for `to_dict`, `from_dict`, `save_network`, `load_network`, `psycopg`).

## Topics to Explore

- [file] `reasons/import_agent.py` — Contains `_sync_claims` (line 336) and its only two call sites (582, 599); the body between 336 and 582 will show whether the function is internally atomic (pre-flight validation, try/except, or rebuild) or leaves partial writes
- [function] `reasons/import_agent.py:_sync_claims` — Read the body to confirm whether it commits `add_node`/`add_nogood` calls before any failure point, and whether it does any dry-run pass
- [general] `network-persistence-layer` — Search the repo for `to_dict`/`from_dict`/`load_network`/`save_network`/`psycopg` usage; `Network` itself has no persistence API, and the `pg` optional dependency in `pyproject.toml` implies a separate module owns this
- [general] `import-agent-callers` — Identify the functions enclosing lines 582 and 599 and check whether they catch exceptions, discard the `network`, or reuse it on subsequent attempts
- [file] `reasons/cli.py` — Top-level entry point; determines whether recovery granularity is one `_sync_claims` call, one agent import, or the entire import job (which dictates whether reuse or reload is even possible)

## Beliefs

- `_sync-claims-mutates-network-in-place` — `_sync_claims` takes `network` as its first parameter (`reasons/import_agent.py:336`), and both call sites (lines 582, 599) return its result without constructing a new Network, consistent with in-place mutation.
- `network-lacks-transactional-rollback` — `Network` in `reasons/network.py` exposes no `snapshot`, `restore`, `copy`, `transaction`, `commit`, or `rollback` method; all state changes in `add_node`, `retract`, `add_nogood`, etc. commit eagerly.
- `network-has-no-persistence-api` — The `Network` class has no `save`, `load`, `to_dict`, or `from_dict` methods; any "reload from DB" behavior must be implemented outside `reasons/network.py`.
- `add-node-commits-before-derived-state` — `Network.add_node` assigns `self.nodes[id] = node` before computing Merkle hashes, inheriting access tags, and computing initial truth value, so a downstream failure leaves a partially-initialized node registered.
- `reload-vs-reuse-undetermined` — The observations omit the `_sync_claims` body and its enclosing callers, so whether import code reloads the Network from persistence or reuses the dirty object cannot be determined from the provided material.
