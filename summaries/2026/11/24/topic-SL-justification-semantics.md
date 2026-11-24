---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Clarify how RMS uses SL antecedents/outlist nodes and why filtering dependencies to the current import batch matters.

## How RMS uses SL antecedents and outlist nodes

### The justification data model

`Justification` (referenced in `reasons/__init__.py:20-21`) carries two lists:

- `antecedents: list[str]` — the inlist; **all must be IN** for the justification to hold (`reasons/__init__.py:11`, `:20`).
- `outlist: list[str]` — **all must be OUT** (`reasons/__init__.py:12`, `:21`).

The module docstring states the semantics explicitly (`reasons/__init__.py:11-15`):

> SL (Support List): node is IN iff ALL antecedents (inlist) are IN AND ALL outlist nodes are OUT. The outlist enables non-monotonic reasoning: "believe X unless Y is believed."

So an SL justification is an implication `A₁ ∧ … ∧ Aₙ ∧ ¬O₁ ∧ … ∧ ¬Oₘ → X`. Multiple justifications on a node are OR'd: the node is IN if **any** one justification holds (`tests/test_outlist.py`, `test_alternate_justification_survives_outlist`).

### How the API constructs them (`reasons/api.py`)

`add_node` (`reasons/api.py:300-334`) maps CLI/API arguments onto a `Justification`:

- `sl="a,b"` → `antecedents=["a", "b"]` (`:323`, `:328`).
- `unless="c,d"` → `outlist=["c", "d"]` (`:302`, `:320`).
- `any_mode=True` expands a multi-antecedent SL into one justification **per** antecedent — i.e., OR instead of AND (`:306`, `:324-326`).
- If only `unless` is given (no `sl`), one SL with `antecedents=[]` and the outlist is created (`:332-334`). That is the canonical "default X unless Y" shape — a node that is IN by default, retracted when a defeater goes IN.

The same mechanism is used elsewhere to disable justifications non-monotonically: `reasons/api.py:1331-1393` (defeat a justification by appending a defeater belief to its `outlist`), and the runtime validity check at `reasons/api.py:1590-1592`.

### How the importer builds them (`reasons/import_beliefs.py`)

The importer parses the markdown registry with `parse_beliefs` (`Depends on:` → `depends_on`, `Unless:` → `unless`). After a topological sort so that dependencies are added before dependents, `import_into_network` builds **one SL justification per claim**:

```python
deps_in_network   = [d for d in claim["depends_on"]      if d in claim_by_id]
unless_in_network = [u for u in claim.get("unless", []) if u in claim_by_id]
if deps_in_network or unless_in_network:
    justifications = [Justification(
        type="SL",
        antecedents=deps_in_network,
        outlist=unless_in_network,
        label=f"imported from beliefs: {claim['type']}",
    )]
```

If neither list ends up non-empty, `justifications` stays `None`, and `network.add_node` treats the claim as a premise (`tests/test_import_beliefs.py::test_imports_premises` asserts `justifications == []`).

Note two important details:

1. **One justification per claim**, not one per antecedent — so `Depends on: a, b` becomes a single AND-justification `SL([a,b])`, not two OR-justifications. There is no `any_mode` at import time.
2. **The filter is against `claim_by_id`, not `network.nodes`.** Despite the variable name `deps_in_network`, the check is `d in claim_by_id` — i.e., *is this dependency in the current registry being imported?* Not "is this dependency in the whole RMS?" That is the "current import batch" the question refers to: the parsed `claims` list from this one `beliefs.md`.

### Why filtering to the current import batch matters

The filter is what makes a partial, self-contained registry importable without corrupting truth values:

1. **Missing antecedents invalidate an SL justification.** SL requires every antecedent to be IN (`reasons/__init__.py:11`). If a claim references a node that will never be added to the network, that reference is "not IN" and the justification fails, forcing the claim OUT. Without the filter, importing a subset of a larger knowledge base would spuriously retract claims that are marked `[IN]` in the file. The topological-sort comment in `import_into_network` codifies the same assumption: a claim is ready when its deps are "already added **or not present in this registry (external dependency)**."

2. **Grounding claims instead of dangling edges.** When a claim has only external deps (nothing in `claim_by_id`), `justifications` stays `None`, so the node is imported as a *premise* — IN by default. That turns an unresolved reference into a conservative assumption ("treat the external dependency as true") rather than a silent retraction. This is also why `tests/test_import_beliefs.py::test_import_real_registry` can assert `in_count > out_count` for a real registry that surely references nodes outside the file.

3. **Out-of-batch defeaters are already inert, but they're also silent.** A missing outlist node is treated as OUT — see `tests/test_outlist.py::test_outlist_node_not_in_network` and `test_outlist_valid_when_out` — so a dangling `unless:` reference never blocks anything. Filtering makes this explicit and avoids recording edges that the runtime will treat as satisfied vacuously.

4. **Determinism and idempotency.** Because edges never cross batch boundaries, importing the same file twice yields the same graph (`test_skip_duplicates`), and a re-import cannot suddenly wire up edges to nodes that came from an unrelated prior import. Cross-registry linkage is handled by the *namespacing* import path (`reasons/api.py:1932`, `reasons/import_agent.py`), not by this local importer.

5. **Reverse index integrity.** `network.add_node` registers dependents so cascades can propagate; `Storage.load` rebuilds them via `network._rebuild_dependents()` (`reasons/storage.py`). Only real intra-batch edges should feed that index, or retracting an unrelated node elsewhere in the network could cascade unexpectedly into this batch. `tests/test_import_beliefs.py::test_dependents_registered` and `test_cascading_works_after_import` exercise the intended intra-batch cascade.

6. **Consistency between sort and edges.** The topological sort decides node insertion order based on which deps exist in `claim_by_id`; the justification builder uses the exact same predicate. Keeping them aligned means a node is only ever wired to a node that will actually be present by the time it is evaluated.

### What the observations don't show

The runtime evaluation that actually enforces "SL antecedents must be IN" is not included — only the outlist half (`outlist_ok` at `reasons/api.py:1590-1592`) and the defeat helper (`:1331-1393`). To fully confirm point (1), the `reasons/network.py` implementation of the justification check (and its treatment of a missing antecedent vs. a missing outlist node) should be read. The outlist behavior is well-covered by `tests/test_outlist.py`; the antecedent-missing case is only implied.

## Topics to Explore

- [file] `reasons/network.py` — Contains the actual SL evaluation and dependency propagation that the observations only summarize; confirms how a missing antecedent vs. a missing outlist node is treated.
- [function] `reasons/import_beliefs.py:import_into_network` — The topological sort plus `claim_by_id` filter that defines what "current import batch" means and why one justification per claim is built.
- [function] `reasons/api.py:add_node` — Source of truth for how `sl`, `cp`, `unless`, and `any_mode` are turned into `Justification` objects; shows the `any_mode` OR-expansion that the importer does **not** use.
- [file] `tests/test_outlist.py` — Executable spec for outlist semantics (missing node = OUT, alternative justification survival, cascading, persistence).
- [general] `namespacing-and-cross-registry-dependencies` — Contrast the local-batch filter here with the namespaced cross-agent import path (`reasons/api.py:1932`, `reasons/import_agent.py`) to see how external deps *are* meant to be linked.

## Beliefs

- `sl-justification-requires-all-antecedents-in-and-all-outlist-out` — An SL justification holds iff every antecedent is IN and every outlist node is OUT (`reasons/__init__.py:11-12`).
- `missing-outlist-node-is-treated-as-out` — An outlist reference to a node absent from the network is treated as OUT, so the default justification still holds (`tests/test_outlist.py::test_outlist_node_not_in_network`).
- `import-filters-dependencies-to-claim-by-id` — `import_into_network` only builds antecedent/outlist edges for dependencies present in the current registry (`d in claim_by_id`), not for nodes elsewhere in `network.nodes`.
- `external-only-dependencies-become-premises` — A claim whose `depends_on` and `unless` are all out of batch is imported with `justifications=None`, i.e., as a grounded premise.
- `importer-produces-one-sl-justification-per-claim` — Unlike `add_node(any_mode=True)`, the importer emits a single SL justification combining all in-batch antecedents rather than one justification per antecedent.
