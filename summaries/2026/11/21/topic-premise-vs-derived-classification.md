---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: How `api.add_node` and the `sl` parameter distinguish premises from derived beliefs, since that classification determines the entire test fixture.

# How `api.add_node` and `sl` distinguish premises from derived beliefs

## Short answer

There is **no explicit premise flag**. The classification is purely structural: a node with no `sl` argument is a premise; a node with a non-empty `sl` argument is a derived belief. `sl` is a comma-separated list of node IDs that justify the new belief.

## Evidence from the observations

### 1. `sl` is a parameter, not a function

`find_sl_symbol` returns zero definitions — `sl` is not a standalone symbol. Its meaning has to be read from `add_node`'s signature. `reasons/pg.py:215` shows the parallel implementation:

```python
def add_node(self, node_id, text, sl="", cp="", unless="", label="", ...)
```

This is the only place in the gathered observations where `sl` appears as a formal parameter. Critically, **`sl` defaults to `""`** — so it can be omitted entirely, and when omitted there is no support list.

`api.add_node` itself is declared at `reasons/api.py:277`, but the observations only returned lines 1–200 of that 5,886-line file. The body is not visible.

### 2. Callers encode the premise/derived split through `sl`

Every test call site follows one of two shapes. Premises are seeded with no `sl`:

```python
api.add_node("a", "A", db_path=db)              # premise
```

and derived beliefs are seeded with a non-empty `sl` naming their supports:

```python
api.add_node("b", "Derived B", sl="a", db_path=db)                 # tests/test_add_justification.py:149
api.add_node("x", "Conclusion", sl="a,b,c", any_mode=True, db_path=db)  # tests/test_any_mode.py:22
```

- `sl="a"` → single-support justified belief.
- `sl="a,b,c"` → multi-support justification whose default semantics are conjunctive (all supports must hold), flipped by `any_mode=True` to disjunctive (any one suffices), as shown across `tests/test_any_mode.py`.
- `unless="enemy"` (`tests/test_any_mode.py:63`) adds a negative/exception support alongside the positive `sl`.

### 3. No explicit premise/derived type exists

`find_derived_belief_symbol` searched for names matching `derived|is_premise|premise` and returned **zero matches**. Combined with the fact that `sl` has no definition of its own, this confirms the model treats premise-vs-derived as an *emergent* property of the justification graph, not a stored discriminator. The `Justification` import at the top of `api.py` (`from . import Justification`) suggests `sl`'s contents get wrapped into a `Justification` object when non-empty.

## Net model

```
add_node(id, text)                  → premise         (no Justification)
add_node(id, text, sl="a")          → derived belief  (Justification: a ⇒ id)
add_node(id, text, sl="a,b", any_mode=True) → derived, disjunctive
add_node(id, text, sl="a", unless="c")      → derived with negative support
```

The whole test fixture relies on this convention: seed the axioms with `add_node`, then build the inference chain with `sl=`. Because premise-ness is inferred rather than declared, the *presence/absence of the `sl` keyword argument at each call site is the classifier*.

## What's missing from the observations

- The actual body of `api.add_node` from `reasons/api.py:277` onward is **not** in the returned slice (only lines 1–200 were fetched). We cannot directly confirm that an empty `sl` skips creating a `Justification`.
- The grep `\bsl\b` against `api.py` returned zero matches even though `add_node` is at line 277 — almost certainly an artifact of the partial read rather than a real absence.
- Semantics of `cp` (likely "consequent" or "called-part"?), and how `unless` and `any_mode` are encoded on `Justification`, are unverified.

## Topics to Explore

- [function] `reasons/api.py:add_node` — Full body starting at line 277; the only place to confirm that empty `sl` short-circuits justification creation
- [file] `reasons/justification.py` — The `Justification` class imported at the top of `api.py`; shows how `sl`/`cp`/`unless` are actually stored
- [file] `tests/test_add_justification.py` — Canonical premise-vs-derived fixture; every line demonstrates the convention
- [file] `reasons/pg.py` — Parallel `add_node` implementation (line 215) and its Python-side validation of `sl`
- [general] `sl-vs-cp-vs-unless` — Precise roles of the three support-list parameters in the belief semantics and how they combine under `any_mode`

## Beliefs

- `sl-defaults-to-empty-string` — `add_node` declares `sl=""`, so omitting the argument is equivalent to passing an empty support list.
- `empty-sl-means-premise` — A node created without `sl` receives no justification and is treated as a premise/root belief.
- `non-empty-sl-produces-derived-belief` — Passing `sl="a"` creates a justification tying the new node to node `a`, making it a derived belief.
- `sl-is-comma-separated-string-of-node-ids` — Multiple supports are encoded in one `sl` string (e.g., `"a,b,c"`), parsed by `add_node`.
- `no-explicit-premise-flag-exists` — No `is_premise`/`derived` symbol exists in the searched scope; premise/derived is inferred from the presence of `sl`, not stored as a flag.
