---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: How `{namespace}:active` is used as a global retraction lever for a whole namespace, and what the intended OR-mode semantics should be.

# How `{namespace}:active` Is Used as a Global Retraction Lever, and What OR-Mode Should Be

## 1. The `{namespace}:active` premise node

A namespace is just a prefix on node IDs, and each namespace is anchored by a **premise node** whose ID is `{namespace}:active`.

- `reasons/api.py:218` (`ensure_namespace_active` docstring) and `reasons/api.py:229` (`active_id = f"{namespace}:active"`) show that the API lazily creates this premise node when a namespace is first used.
- `reasons/api.py:246–257` shows how namespaces are *discovered*: it scans nodes, matches those whose ID ends with `":active"` and whose `metadata["role"] == "agent_premise"`, then strips the `:active` suffix to recover the namespace prefix (`ns = nid[:-len(":active")]`).
- `reasons/pg.py:2206` mirrors the same "look for `%:active`" pattern in the Postgres backend, and `reasons/pg.py:2215` strips the suffix the same way.
- `reasons/derive.py:150–151` explicitly *skips* any node whose ID ends in `:active` during derivation — they're control-flow premises, not content.

So `{namespace}:active` is a single, well-known premise node per namespace.

## 2. How it becomes the “global retraction lever”

The lever works because every node created *inside* the namespace is wired to depend on that premise:

- `reasons/api.py:305` — the `namespace` parameter docs say “auto-creates `ns:active` premise”.
- `reasons/api.py:337` — *Namespace support: prefix node_id and add dependency on ns:active*.
- `reasons/api.py:350` — *Add `ns:active` as antecedent to the justification* (for nodes that already have an explicit SL).
- `reasons/api.py:357` — *No explicit justification — create an SL depending on `ns:active`* (for nodes with no other support).

Because `ns:active` is an antecedent of every justification in the namespace, **retracting `ns:active` causes every one of those justifications to lose support, and the propagation engine cascades the whole namespace OUT**. That is the "retraction lever": one `retract ns:active` invocation toggles the entire namespace at once.

`reasons/api.py:1938` documents exactly this intent for the agent flavor:

> *node `'agent_name:active'`. Retracting that premise cascades OUT all …*

## 3. The agent-specific variant (outlist-based lever)

`reasons/import_agent.py:4–9` describes a subtle twist: for imported agents the `:active` premise is **not** placed directly in the antecedents of every belief (line 9). Instead:

- `import_agent.py:41–45` creates both `agent_name:active` and an `agent_name:inactive` companion.
- Line 6 says `agent:active`/`agent:inactive` interact such that retracting `agent:active` makes `agent:inactive` go IN, which then cascades OUT every belief. This is the standard defeasible pattern of putting `agent:inactive` (or the negation of `active`) in the **outlist** of imported beliefs, so the `:active` premise acts as the gate without being a conjunction antecedent.
- `import_agent.py:533` additionally mentions a *relay node* created alongside `agent_name:active`.

So: for plain namespaces the lever is *antecedent-based*; for agent imports the lever is *outlist/relay-based*. Both achieve the same "one retraction knocks the whole namespace OUT" effect.

## 4. Intended OR-mode semantics (`any_mode`)

OR-mode is exposed as `any_mode` in `reasons/api.py` and `--any` on the CLI (`reasons/cli.py:35–80`, `reasons/derive.py:664`).

**Intended semantics** (from `tests/test_any_mode.py`):

1. **Default SL is conjunctive.** `tests/test_any_mode.py:86–96` (`test_without_any_single_justification`) confirms that `sl="a,b,c"` without `any_mode` yields *one* justification with *three* antecedents — conclusion is IN only if all three are IN.

2. **`any_mode=True` is disjunctive.** `tests/test_any_mode.py:19–26` (`test_any_creates_separate_justifications`) confirms it expands into *three* SL justifications, each with *one* antecedent. The conclusion is IN if *any* antecedent is IN — classic OR semantics.

3. **Retraction behaviour:**
   - Survives single-premise retraction — `test_any_survives_single_retraction` (line 34).
   - Goes OUT only when **all** branches are retracted — `test_any_goes_out_when_all_retracted` (line 43).
   - Single-premise SL is not expanded — `test_any_with_single_premise_no_expansion` (line 53).

4. **Outlists are replicated on each branch**, so the disjunction is still blocked by its defeaters: `test_any_preserves_outlist` (line 63). Each expanded justification carries `enemy` in its outlist.

5. **Add-justification can restore an OUT node** — `test_any_add_justification_restores_out_node` (line 107) shows `result["new_truth_value"] == "IN"` after adding an `any_mode` justification.

6. **Premise-count reporting uses the expanded shape** — `test_add_any_returns_low_premise_count` returns `1`, and the `--any` flag suppresses the 3+ premise warning at `reasons/cli.py:35–37`.

7. **SQLite only.** `reasons/api.py:313–314` and `reasons/api.py:432–433` raise `NotImplementedError("any_mode is not supported with PostgreSQL")`. The Postgres backend at `reasons/pg.py` has no equivalent expansion.

**Implementation points visible in the observations:**

- `reasons/api.py:324` — `if any_mode and len(antecedents) > 1:` in `add_node` triggers the expansion.
- `reasons/api.py:457` — `if any_mode and jtype == "SL" and len(antecedents) > 1:` is the `add_justification` variant.
- Because `add_node`’s any_mode branch (`:324`) runs **before** the namespace branch (`:337`), the `ns:active` antecedent is appended to *each* expanded branch. That preserves the global retraction lever: retracting `ns:active` still kills every OR branch simultaneously, so OR-mode does not accidentally “escape” a namespace you retracted.

That last point is the intended interaction: **`any_mode` gives you OR between content premises, while `{namespace}:active` remains a mandatory AND-ed condition across every OR branch.**

## 5. What's missing from these observations

- The full bodies of `add_node` (around `reasons/api.py:300–360`) and `add_justification` (`:410–470`) are not shown, so the exact order of *expansion → namespace-antecedent injection → save* is inferred from line numbers rather than read.
- `reasons/import_agent.py:533` mentions a "relay node" but the implementation is not shown; the precise agent cascade (inactive/outlist/relay interaction) cannot be verified beyond lines 4–9 and 41–45.
- There is no test file shown that combines `any_mode=True` **with** a `namespace` argument, so the two features’ joint behaviour is only inferred from line ordering — it is not test-verified in the observations.
- The SQL that populates `role == "agent_premise"` metadata (used at `reasons/api.py:256`) is not visible.

## Topics to Explore

- [function] `reasons/api.py:add_node` — Read lines ~300–360 in full to verify the exact ordering of `any_mode` expansion and `ns:active` antecedent injection, and confirm OR branches all carry the namespace lever.
- [file] `reasons/import_agent.py` — Inspect the relay-node creation around line 533 and the `agent:active` / `agent:inactive` wiring to see how the outlist-based lever is actually constructed.
- [function] `reasons/derive.py:propagate` — See how the engine walks justifications and treats `:active` nodes differently, which is what makes single-retraction cascades propagate namespace-wide.
- [general] `any-mode-postgres-gap` — Track down where `reasons/pg.py` would need the equivalent per-antecedent expansion to lift the `NotImplementedError` at `api.py:313` and `api.py:432`.
- [general] `namespace-any-mode-interaction` — Look for (or add) a test that combines `namespace=` with `any_mode=True` to lock in the behaviour of retracting `ns:active` across OR branches.

## Beliefs

- `namespace-active-enables-global-retraction` — Because `ns:active` is added as an antecedent to every justification in the namespace (`reasons/api.py:350`, `:357`), retracting it cascades the entire namespace OUT.
- `any-mode-implements-or-semantics` — `any_mode=True` expands a multi-antecedent SL into one justification per antecedent (`reasons/api.py:324`, `:457`), so the conclusion survives retraction of any single premise.
- `namespace-active-is-skipped-in-derivation` — `reasons/derive.py:150–151` explicitly skips nodes whose ID ends with `:active`, so the lever is invisible to normal derivation logic.
- `any-mode-requires-sqlite` — `reasons/api.py:313–314` and `:432–433` raise `NotImplementedError` when `any_mode` is combined with the PostgreSQL backend.
- `agent-inactive-prevents-beliefs` — Per `reasons/import_agent.py:4–9`, retracting `agent:active` drives `agent:inactive` IN (via outlist, not antecedent), which cascades every imported belief OUT.
