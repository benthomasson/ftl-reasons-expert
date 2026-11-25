---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Meta-question: why SL flattening/outlist drop is semantically significant in a TMS, and whether a wire format that preserves full justification structure would be preferable.

# Why SL Flattening / Outlist Drop Matters in a TMS

## The semantic unit is the justification, not the antecedent set

In this codebase a belief node is IN iff **any** of its justifications is valid (`README.md`, `CLAUDE.md`). A `Justification` is a first-class record with three fields:

- `type`: `"SL"` or `"CP"` (`reasons/__init__.py:19`)
- `antecedents` (the **inlist**): nodes that must be IN
- `outlist`: nodes that must be **OUT** (`reasons/__init__.py:21`)

The canonical validity rule is at `reasons/api.py:1590-1592`:

```python
outlist_ok = all(
    ... for o in j.outlist
)
```

combined with the ALL-inlist check documented at `reasons/__init__.py:11-12`. So **"X is IN" is definitionally the pair (inlist, outlist)** — dropping either half changes what the system means by "believed."

---

## Two different "flattenings" — both are lossy, but of different semantics

### 1. Outlist drop (structural flattening)

Collapsing `SL(antecedents=[...], outlist=[...])` into a flat antecedent list erases the non-monotonic half of the justification. This silently changes:

- **Default reasoning.** `reasons add X --unless Y` (`README.md`, `reasons/api.py:320, 437`) produces `Justification(type="SL", antecedents=[], outlist=["y"])`. Strip the outlist and you get `SL(antecedents=[])` — an **unconditional** justification, so `X` stays IN forever. The test that proves this (`test_outlist.py:test_assert_outlist_node_retracts_dependent`) would fail: asserting `Y` must retract `X`.
- **Restoration.** Doyle's key claim in this repo (`CLAUDE.md` design decisions: "Retracted nodes stay in DB … enables restoration without rederivation") depends on the outlist edge being in the dependency graph. `test_outlist.py:test_retract_outlist_node_restores_dependent` requires that retracting `Y` restores `X`. Without the outlist edge, the graph doesn't know `X` depends on `Y`, so restoration never fires.
- **Dialectical argumentation.** `reasons/api.py:1331-1334` (`defeat`) is *defined* as appending a defeater to `justification.outlist` (`api.py:1392-1393`, `1526-1527`). The whole `challenge`/`defend` command surface in `README.md` is implemented as outlist mutation. Drop the outlist and challenge/defend collapse to nothing.
- **Explain traces.** `reasons/api.py:899` serializes `type`, `antecedents`, and `outlist` side-by-side, and `test_outlist.py:TestOutlistExplain` asserts `steps[0]["outlist"]` and `steps[0]["violated_outlist"]`. A flattened stream cannot produce either field.

### 2. SL "any_mode" expansion (conjunction → disjunction)

Separately, `any_mode` at `reasons/api.py:457` rewrites one multi-antecedent SL into *N* single-antecedent SLs:

```python
if any_mode and jtype == "SL" and len(antecedents) > 1:
    for a in antecedents:
        j = Justification(type="SL", antecedents=[a], outlist=outlist, label=label)
```

That converts a strict logical conjunction (`all of a, b, c`) into a disjunction (`any of a, b, c`) — a strictly weaker belief condition. The codebase itself flags this as a real hazard: `reasons/api.py:4186-4188` runs an **LLM review specifically to detect ALL-vs-ANY misclassification**, and `reasons/cli.py:36-38` prints a tip whenever an SL has 3+ premises without `--any`. If a wire format "flattens" a multi-antecedent SL into separate justifications, it performs exactly this potentially unsound transformation *by accident*.

**Both flattenings are semantically significant**, but in opposite directions: outlist drop makes the system too strong (monotonic, never retracts); any_mode expansion makes it too weak (disjunctive, over-retracts).

---

## Would a wire format that preserves full justification structure be preferable?

**Yes — and the code is already half-committed to it, which is why the gap is worth closing.**

Evidence for the "preserve structure" intent:

- `reasons/api.py:899` writes `{"type": j.type, "antecedents": j.antecedents, "outlist": j.outlist, ...}` — the schema carries all three semantic fields.
- `CLAUDE.md` lists `reasons import-json network.json` as "lossless" and advertises `test_import_json.py # JSON round-trip`.
- `reasons/api.py:2463` checks `j.get("type") == "SL" and j.get("antecedents")` — deserialization reads both fields.

But several things are concerning:

1. **`search_serialization_helpers` returned only 1 match** (and it's a docstring in `reasons/repair.py:5`). That means there is **no single canonical `to_dict`/`from_dict`/`serialize`**; serialization is inlined at `reasons/api.py:899` and deserialization is inlined at `api.py:2568` / `api.py:2639` via `.get(...)`. Without a single contract, every new writer/reader is an opportunity to drop a field.

2. **Defaults silently mask loss.** Deserialization uses `jdata.get("type", "SL")` (`api.py:2568, 2639`) — reasonable since SL is common — but if the reader also does `jdata.get("outlist", [])`, an omitted `outlist` becomes an empty list, converting a non-monotonic justification into a monotonic one **with no error**. Combined with the dataclass default at `reasons/__init__.py:21` (`outlist: list[str] = field(default_factory=list)`), the "drop outlist" failure mode is invisible: the node still validates, it just never goes OUT when it should. This is the exact silent-downgrade path the meta-question is about.

3. **The AI review at `reasons/api.py:4186-4188` is a tell.** If the codebase already needs an LLM to audit whether an SL was written with ALL vs ANY intent, it clearly cannot rely on post-hoc interpretation of a flattened wire format. The justification structure is the only place that intent survives; flattening at the boundary forfeits it.

**Verdict:** A wire format that keeps `{type, antecedents, outlist}` verbatim (as line 899 already does) is the right design. What's missing is (a) enforcing that as the *only* serializer/deserializer, (b) making absent `outlist` a loud error rather than a silent `[]`, and (c) doing the same for `type` and the ALL-vs-ANY distinction so that `any_mode` is an explicit caller decision rather than an incidental consequence of format conversion.

---

## What the observations don't show

- The exact body of `from_dict` at `reasons/api.py:2568` and `2639` — I can't confirm whether `outlist` defaults to `[]` or raises.
- Whether `reasons/merkle.py` (mentioned in `CLAUDE.md`) hashes `outlist` alongside antecedents — if it hashes only the antecedent set, then outlist mutations (e.g., a `defeat`) would evade integrity checks.
- Whether `reasons import-json` (`CLAUDE.md`) round-trips `outlist` losslessly — `test_import_json.py` exists but wasn't included.
- Whether any code path (e.g., wiki/compact output at `reasons/build_wiki.py:174`, `reasons/api.py:2925`) writes a "flat beliefs.md" and thereby drops outlists on re-import.

## Topics to Explore

- [file] `reasons/api.py` — Read `add` (lines 300–334), `defeat` (1331–1393), the outlist validation at 1590–1592, and the JSON writer at 899 together; this is where the semantic unit is built, mutated, and serialized
- [file] `reasons/__init__.py` — The `Justification` dataclass at line 7 is the entire semantic contract; note the `outlist` field default at line 21
- [function] `reasons/api.py:1590` — The actual justification-validity predicate combining `inlist` and `outlist`; every flattening argument must hold up against this function
- [general] `any-mode-all-vs-any` — The disjunction-vs-conjunction hazard guarded by the LLM review at `reasons/api.py:4186-4188` and the CLI tip at `reasons/cli.py:36-38`
- [diff] `import-json-roundtrip` — Verify that `test_import_json.py` proves outlist and type survive export/import, and whether `merkle.py` covers outlist mutations

## Beliefs

- `justification-is-the-semantic-unit` — A node is IN iff any of its `Justification` records is valid; validity is defined over the (type, antecedents, outlist) triple, not over the node's ID set
- `outlist-enables-nonmonotonic-reasoning` — A `Justification` holds iff all `antecedents` are IN **and** all `outlist` nodes are OUT (`reasons/__init__.py:11-12`, `reasons/api.py:1590-1592`)
- `outlist-drop-breaks-cascade-and-restoration` — Removing `outlist` from a serialized justification makes it monotonic, so asserting the defeater no longer retracts the dependent and retracting it no longer restores it (`tests/test_outlist.py:test_assert_outlist_node_retracts_dependent`, `test_retract_outlist_node_restores_dependent`)
- `any-mode-flattens-conjunction-to-disjunction` — `any_mode` at `reasons/api.py:457` expands one multi-antecedent SL into N single-antecedent SLs, replacing ALL semantics with ANY semantics
- `from-dict-defaults-mask-loss` — Deserialization at `reasons/api.py:2568` and `2639` uses `jdata.get(..., default)`, so a wire format that omits `type` or `outlist` silently falls back to `"SL"` and `[]` rather than failing loudly
- `justification-serialization-is-inlined` — There is no canonical `to_dict`/`from_dict` helper (only 1 match in `search_serialization_helpers`); serialization lives inline at `reasons/api.py:899` and deserialization is scattered across `api.py:2463`, `2568`, and `2639`
