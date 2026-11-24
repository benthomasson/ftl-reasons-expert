---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: `SKILL.md:38` introduces nogoods (a set of nodes that cannot all be IN); check whether the resolution machinery follows Doyle's contradiction handling or de Kleer-style nogood records, since the codebase cites only Doyle.

## Short answer

The codebase is best described as **Doyle-style contradiction resolution with a persistent nogood ledger**, not as a de Kleer-style ATMS. Nogoods are first-class records, but the resolution machinery is described in terms of **dependency-directed backtracking** and **entrenchment scoring**, which are Doyle/JTMS concepts. The codebase’s observed citations are Doyle-only; there are no de Kleer citations in the gathered matches.

The exact text of `SKILL.md:38` is not in the observations, but the same definition appears in `README.md:13`: a nogood is “a set of nodes that cannot all be IN simultaneously.” The more important lines are `SKILL.md:99` and `CLAUDE.md:16`, which say that recording a nogood triggers dependency-directed backtracking and entrenchment-based selection of the responsible premise.

## What the observations show

### Nogoods are real, persisted records

- `reasons/api.py:1759` defines `add_nogood`.
- The documented return contract at `reasons/api.py:1763` is:
  - `nogood_id`
  - `nodes`
  - `changed`
  - `backtracked_to`
- The local implementation path calls `net.add_nogood(node_ids)` and then reads `net.nogoods[-1]` (`reasons/api.py:1778-1781`). This implies `Network.add_nogood` mutates the network and appends a `Nogood` object.
- `tests/test_nogood_id.py` verifies content-derived IDs such as `nogood-a-b`, duplicate suffixes like `nogood-a-b-2`, and persistence across save/load.
- `reasons/api.py:1879-1881` includes nogoods in network export.
- `reasons/api.py:1890-1917` imports nogoods from JSON/Markdown, including auto-detected `nogoods.md`.

So the system does maintain explicit nogood records, which is the de Kleer-ish part.

### But resolution is Doyle-style, not ATMS-style

The key evidence for Doyle-style handling is:

- `CLAUDE.md:16`: “When a nogood is detected, traces backward to find the responsible premise using entrenchment scoring.”
- `SKILL.md:99`: recording a contradiction “uses dependency-directed backtracking to trace” the responsible premise.
- `SKILL.md:261`: after `nogood`, report “what premise was backtracked to and what cascaded.”
- `CHANGELOG.md:132`: “Dependency-directed backtracking with nogood detection.”
- `reasons/network.py:3`: “Implements Doyle’s (1979) TMS algorithm.”
- `README.md:3`, `CLAUDE.md:5`, `reasons/__init__.py:1`, and `README.md:198` all cite Doyle.

This is not how a de Kleer-style ATMS primarily uses nogoods. In an ATMS, nogoods are used to compute and maintain **consistent assumption environments/context labels**. The observed API instead returns `changed` and `backtracked_to`, and the skill docs say a premise is backtracked to and cascades follow. That is a Doyle/JTMS-style global retraction/restoration mechanism.

### `contradictions.py` detects candidates; it does not implement ATMS context resolution

`reasons/contradictions.py` sends batches of IN beliefs to an LLM and parses responses like:

```text
### NOGOOD short-kebab-id
- Claims: belief-id-1, belief-id-2
- Analysis: ...
- Severity: High|Medium|Low
```

The parser is `parse_contradiction_response`. The tests in `tests/test_contradictions.py` verify parsing, filtering, batching, and that `auto_apply` calls `add_nogood`. That module is a **detection/proposal** layer. It does not appear to maintain de Kleer-style environment labels or contexts.

### `repair.py` is a separate repair path

The observed portion of `reasons/repair.py` triages invalid derived beliefs into:

- `search_and_link`
- `soften`
- `abandon`
- `research`

It focuses on review-invalid beliefs and “smuggled premises,” not on nogood-driven context resolution. The observed excerpt contains no nogood handling. So the repair machinery is not evidence of a de Kleer-style nogood resolver either.

## Classification

| Aspect | Observed behavior | Interpretation |
|---|---|---|
| Nogood storage | `Nogood` records with IDs, persistence, import/export | de Kleer-ish record keeping |
| Contradiction detection | LLM proposes sets of incompatible IN nodes | detection layer |
| Resolution | Dependency-directed backtracking, entrenchment scoring, retraction/cascade | Doyle/JTMS-style |
| Multiple contexts / assumption labels | No evidence in observations | not de Kleer ATMS |
| Citations | Doyle 1979 only | codebase self-identifies as Doyle |

So the most accurate statement is:

> The system follows Doyle’s contradiction-handling model, augmented with explicit persistent nogood records. The nogood record is a stored explanation/audit artifact and trigger, but the actual resolution path is dependency-directed backtracking with entrenchment-based premise retraction, not de Kleer-style ATMS nogood/context labeling.

The main missing evidence is the implementation of `Network.add_nogood` in `reasons/network.py`. The observations do not include that function body, so we cannot inspect the exact entrenchment scoring or backtracking algorithm. But `CLAUDE.md:16`, `SKILL.md:99`, `SKILL.md:261`, and `CHANGELOG.md:132` are strong enough to classify the intended behavior.

## Topics to Explore

- [function] `reasons/network.py:Network.add_nogood` — The exact backtracking and entrenchment logic is not in the observations; this is the core resolution implementation.
- [file] `reasons/network.py` — Implements the Doyle TMS propagation/retraction/restoration model referenced at line 3.
- [file] `reasons/api.py` — Shows how `add_nogood`, detection, import/export, and return contracts connect the CLI/API to the network.
- [file] `reasons/contradictions.py` — Generates nogood candidates via LLM prompts and parsing, separate from actual resolution.
- [general] `de-kleer-atms-vs-doyle-jtms` — Useful comparison for why persistent nogoods alone do not make this an ATMS.

## Beliefs

- `nogoods-are-persisted-with-content-ids` — `Nogood` IDs are derived from sorted node IDs, e.g. `nogood-a-b`, and survive save/load.
- `add-nogood-documented-contract-includes-backtracked-to` — `api.add_nogood` documents return fields `nogood_id`, `nodes`, `changed`, and `backtracked_to`.
- `contradiction-auto-apply-calls-add-nogood` — `api.detect_contradictions(auto_apply=True)` invokes `add_nogood` for detected contradictions.
- `repair-module-triages-invalid-beliefs` — The observed `repair.py` code triages invalid beliefs into `search_and_link`, `soften`, `abandon`, or `research`.
- `codebase-citations-are-doyle-only` — The observed Doyle/de Kleer citation search found Doyle references but no de Kleer references.
