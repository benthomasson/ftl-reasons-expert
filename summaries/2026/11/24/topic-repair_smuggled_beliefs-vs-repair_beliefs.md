---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Resolve the naming mismatch between the test import (line 10) and the orchestrator defined in `reasons/repair.py`; this is the most likely candidate for a sealed regression.

## What the observations show

There are two competing names for the same orchestrator, and the test suite references the one that isn't documented:

| Where | Line | Name |
|---|---|---|
| `tests/test_repair.py` | 10–15 (import block starts line 10) | `repair_smuggled_beliefs` |
| `reasons/repair.py` | 9 (module docstring) | `repair_beliefs` |
| `reasons/repair.py` | 1–250 | *no* `def repair_*` orchestrator yet (only `parse_extract_response`, `parse_match_response`, `extract_smuggled_claim`, `find_matching_premises`, `parse_triage_response`, `parse_soften_response`, `triage_belief`) |

The import block in `tests/test_repair.py` is:

```python
from reasons.repair import (
    parse_extract_response,
    parse_match_response,
    extract_smuggled_claim,
    find_matching_premises,
    repair_smuggled_beliefs,     # <-- only name not accounted for
)
```

Four of those five names are confirmed present in the first 250 lines of `reasons/repair.py`. The fifth — `repair_smuggled_beliefs` — is the only unverified symbol. Meanwhile `reasons/repair.py:9` explicitly states:

> "The `repair_beliefs` orchestrator triages each invalid belief via LLM, then executes the appropriate pattern."

and `grep_orchestrator_references` (pattern `orchestrat`) returns **exactly one** hit — that same docstring line. So there is no symbol literally named `*orchestrat*`; the orchestrator's canonical name per the module's own contract is `repair_beliefs`.

## Why this is the "sealed regression" candidate

1. **It fails at collection time, not assertion time.** A `from … import` of a nonexistent name raises `ImportError` when pytest imports `tests/test_repair.py`. Nothing in the 542-line module runs — every test in `TestParseExtractResponse` and everything below it is silently "sealed" behind one error line. That is exactly the shape of a regression that hides itself: no assertion output, just a collection error attributed to line 10.
2. **It is not isolated to one file.** `reasons_lib/repair.py` is a three-line shim: `from reasons.repair import *`. It has no `__all__` re-export of its own visible in the excerpt, so the canonical spelling in `reasons/repair.py` determines what `reasons_lib.repair` exposes to downstream callers. Renaming or aliasing the orchestrator ripples through that path.
3. **A sibling test module shares the same import shape.** `tests/test_research.py:10` also does `from reasons.repair import (`. If it pulls the same orchestrator symbol, one root cause seals *two* modules.
4. **The surrounding evidence points to a rename.** The module docstring still says "Three patterns" while `VALID_PATTERNS` already includes `"research"` (four patterns) — the module has been generalized from "repair smuggled beliefs" toward "repair beliefs" (link / soften / abandon / research). A rename `repair_smuggled_beliefs` → `repair_beliefs` would be the natural companion of that generalization, and `tests/test_repair.py` was left behind while `tests/test_research.py` was added.

## What is missing before you can pick the fix

The excerpt of `reasons/repair.py` stops at line 250 of 618. The actual `def` statement for the orchestrator is **not in the returned range**, so we are inferring the name from the line-9 docstring rather than seeing it. Two hypotheses remain live and need exactly one grep to separate:

- **H1 (docstring is right):** the orchestrator is `def repair_beliefs(...)` → the test import is stale; fix the test (and any other stale import).
- **H2 (docstring is stale):** the orchestrator is `def repair_smuggled_beliefs(...)` → the docstring's prose is wrong and the test is correct; fix the docstring.

Also unconfirmed: whether `reasons/repair.py` defines `__all__`, what `tests/test_research.py:10` actually imports, and whether an alias already exists further down the file.

## Resolution procedure

1. `grep -n "^def repair" reasons/repair.py` (or read lines 250–618) to get the real symbol name. Confirm its signature and return contract while you're there.
2. If H1: update `tests/test_repair.py:10-15` to `repair_beliefs`, and check `tests/test_research.py:10` for the same stale name.
3. If H2: correct `reasons/repair.py:9` and the `TRIAGE_PROMPT` docs, and leave tests untouched.
4. **In either case, add a backwards-compatible alias** rather than a bare rename:
   ```python
   repair_smuggled_beliefs = repair_beliefs  # deprecated alias
   ```
   This keeps the `from reasons.repair import *` shim in `reasons_lib/repair.py` working for external consumers, and keeps both spellings importable so the collection error cannot re-seal. If you add `__all__` to `reasons/repair.py` while doing this, include both names so the shim keeps exporting them.
5. Re-run `pytest tests/test_repair.py tests/test_research.py` and confirm collection succeeds — the real assertion failures, if any, are currently invisible behind the import error.

## Topics to Explore

- [file] `reasons/repair.py` — Read lines 250–618 to confirm the actual `def` name of the orchestrator, its signature, and whether it returns a summary dict or mutates node state in place.
- [file] `tests/test_research.py` — Check the line-10 import block: if it requests the same orchestrator symbol, the same root cause seals a second module.
- [file] `reasons_lib/repair.py` — The `from reasons.repair import *` shim; determine whether `reasons/repair.py` defines `__all__`, since its absence means every public top-level name is re-exported and the canonical name matters to external callers.
- [general] `repair-orchestrator-rename-history` — A git log/blame over `reasons/repair.py` and `tests/test_repair.py` would confirm whether `repair_smuggled_beliefs` was renamed when the `research` pattern was added.
- [general] `three-vs-four-patterns-docstring-drift` — The docstring at `reasons/repair.py:1` says three patterns while `VALID_PATTERNS` has four; auditing other doc/code drift in this module may surface related stale references.

## Beliefs

- `test-repair-imports-repair-smuggled-beliefs` — `tests/test_repair.py` (import block starting at line 10) requests `repair_smuggled_beliefs` from `reasons.repair`.
- `repair-py-names-orchestrator-repair-beliefs` — `reasons/repair.py:9` documents the orchestrator as `repair_beliefs`, and no other `*orchestrat*` reference exists in the codebase.
- `reasons-lib-repair-is-a-star-shim` — `reasons_lib/repair.py` is exactly one line, `from reasons.repair import *`, so its public surface is determined entirely by `reasons/repair.py`'s top-level names.
- `missing-import-name-fails-whole-module-collection` — An `ImportError` on `tests/test_repair.py:10` prevents any of the module's 542 lines of tests from executing, converting assertion failures into a single collection error.
- `valid-patterns-includes-research` — `VALID_PATTERNS` in `reasons/repair.py` contains `{"search_and_link", "soften", "abandon", "research"}`, four patterns, contradicting the "Three patterns" module docstring.
- `parse-helpers-confirmed-present-in-repair-py` — `parse_extract_response`, `parse_match_response`, `extract_smuggled_claim`, and `find_matching_premises` all appear within `reasons/repair.py` lines 1–250, leaving only the orchestrator symbol unverified.
