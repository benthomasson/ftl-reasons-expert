---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Explore how `type="CP"` is used (hypothetical reasoning / disjunctive belief) versus `type="SL"` and whether CP justifications carry outlists.

## How `type="CP"` compares with `type="SL"`

The `Justification` dataclass defines `type` as a string documented as `"SL"` or `"CP"` and defines `outlist` as a normal field on every justification, independent of type (`reasons/__init__.py:7`, `reasons/__init__.py:19`, `reasons/__init__.py:21`). So at the model layer, CP is not structurally different from SL with respect to carrying outlists.

### Creation and prevalence

`SL` is clearly the default and dominant justification type:

- Missing persisted types default to `"SL"` when loading justifications (`reasons/api.py:2568`, `reasons/api.py:2639`; also `reasons/import_agent.py:147`).
- Many explicit creation paths build `Justification(type="SL", ...)`, including the main API add-node paths (`reasons/api.py:326`, `328`, `334`), `add_justification` paths (`reasons/api.py:359`, `460`, `1518`), and import/network utilities (`reasons/import_agent.py:62`, `232`; `reasons/network.py:472`, `588`, `795`).
- The grep counts show 241 `"SL"` matches versus only 7 `"CP"` matches.

`CP` is a first-class but much less common alternative:

- In the add-node/add-justification creation area, CP is created with the same `antecedents` and `outlist` arguments as SL:  
  `justifications.append(Justification(type="CP", antecedents=antecedents, outlist=outlist, label=label))` (`reasons/api.py:331`).
- In `add_justification`, `jtype` can be set to `"CP"` (`reasons/api.py:444`) and the final generic construction passes `type=jtype, ..., outlist=outlist` (`reasons/api.py:466`).
- The API test `test_add_cp_justification` confirms that `api.add_justification("b", cp="a")` creates a second justification whose stored `type` is `"CP"` (`tests/test_add_justification.py:154-168`).
- PostgreSQL schema explicitly allows both types: `type TEXT NOT NULL CHECK (type IN ('SL', 'CP'))` (`reasons/pg.py:45`). There is also at least one stored/CO-produced CP justification dictionary in `reasons/pg.py:3683`.

### Do CP justifications carry outlists?

Yes, based on the observed construction and serialization paths:

- `Justification` itself always has an `outlist` field (`reasons/__init__.py:21`).
- The CP construction at `reasons/api.py:331` explicitly passes `outlist=outlist`.
- The generic `add_justification` construction at `reasons/api.py:466` passes `outlist=outlist` for whatever `jtype` was selected, including CP (`jtype = "CP"` at `reasons/api.py:444`).
- Serialization includes `outlist` for every justification, with no CP exclusion: `{"type": j.type, "antecedents": j.antecedents, "outlist": j.outlist, ...}` (`reasons/api.py:899`).
- Network evaluation enters the same branch for both types: `if j.type in ("SL", "CP"):` (`reasons/network.py:982`). That strongly suggests the later antecedent/outlist validation logic is shared, not SL-only.
- There is generic outlist validation code using `j.outlist` at `reasons/api.py:1590`, and no observed type guard excluding CP.

So the data model, API construction, persistence schema, serialization, and the shared network type gate all support CP justifications carrying outlists.

However, there is a coverage gap: the observed outlist tests (`tests/test_outlist.py`) create only `Justification(type="SL", ...)`, and the observed CP test only asserts `type == "CP"` (`tests/test_add_justification.py:167`). There is no observed test that creates a CP justification with a non-empty outlist and verifies the outlist gating behavior. The conclusion that CP outlists are enforced is therefore supported by shared code paths, but not directly proven by a CP-specific test in these observations.

### Hypothetical / disjunctive semantics

The observations do **not** establish that `type="CP"` means “hypothetical reasoning” or “disjunctive belief.” The grep for `hypothetical|disjunctive` does not hit any CP literal near it:

- `reasons/api.py:4189` mentions conjunctive `ALL` versus disjunctive `ANY` in a docstring.
- `reasons/review_justifications.py:4` and `:29` describe `ALL` and `ANY` review categories.
- No observed line links `"CP"` to `hypothetical` or `disjunctive`.

More importantly, the one observed code path that turns multi-antecedent SL justifications into single-antecedent “ANY” style alternatives is explicitly SL-only:

```python
if any_mode and jtype == "SL" and len(antecedents) > 1:
```

at `reasons/api.py:457`. That branch is followed by construction of `Justification(type="SL", antecedents=[a], ...)` at `reasons/api.py:460`. CP is not included in that disjunctive/ANY splitting logic.

Other observed SL-specific special cases also suggest CP is not treated as a variant of SL:

- `reasons/api.py:499`: `if j.type == "SL" and len(j.antecedents) >= 2:`
- `reasons/api.py:2463`: `if j.get("type") == "SL" and j.get("antecedents"):`

Thus, from the observations, the defensible interpretation is:

- `SL` is the default, dominant justification type, with explicit ALL/ANY handling in some paths.
- `CP` is an alternate first-class justification type accepted by the model, API, storage, and network validator.
- The code observed here does not prove that CP means hypothetical/disjunctive reasoning; any such mapping must be confirmed by reading the full network evaluation logic and the full `add_justification`/add-node paths.

What is missing: the full body of `reasons/network.py` around line 982, the full `add_node`/`add_justification` logic around `reasons/api.py:300-466`, and any CP-specific outlist test. Those would determine whether CP antecedents are evaluated as AND, OR, or some special hypothetical construct, and whether CP outlists are definitely enforced.

## Topics to Explore

- [file] `reasons/network.py` — Inspect the full branch beginning at line 982 where `"SL"` and `"CP"` are accepted together; confirm whether antecedent and outlist validation is identical for both types.
- [function] `reasons/api.py:add_justification` — Read the full function around lines 406–466 to confirm CP creation precedence, outlist handling, and why `any_mode` only splits SL justifications.
- [file] `reasons/api.py` — Examine the add-node/justification creation path around lines 302–334, especially the CP branch at line 331 and its `outlist=outlist` argument.
- [file] `reasons/review_justifications.py` — Read the ALL/ANY disjunctive review logic to see whether it is intended to relate to CP or only to SL justifications.
- [file] `tests/test_add_justification.py` — The CP test at lines 154–168 verifies only the type field; check whether any other test covers CP with a non-empty outlist.

## Beliefs

- `cp-justifications-carry-outlist-field` — CP justifications are constructed with `outlist=outlist` at `reasons/api.py:331` and through the generic `type=jtype` construction at `reasons/api.py:466`, where `jtype` can be `"CP"`.
- `network-evaluates-sl-and-cp-together` — `reasons/network.py:982` enters the same branch for justification types `"SL"` and `"CP"`.
- `sl-is-default-justification-type` — Persisted justifications missing a `type` default to `"SL"` at `reasons/api.py:2568`, `reasons/api.py:2639`, and `reasons/import_agent.py:147`.
- `disjunctive-any-mode-is-sl-only` — The `any_mode` multi-antecedent splitting logic is guarded by `jtype == "SL"` at `reasons/api.py:457`, so CP is not handled by that path.
- `pg-schema-allows-sl-and-cp` — The PostgreSQL justification schema constrains `type` to `('SL', 'CP')` at `reasons/pg.py:45`.
