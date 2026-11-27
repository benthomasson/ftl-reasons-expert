---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Compare the silent `_retracted` skip with the warning-and-skip behavior for dangling dependents.

## Comparing `_retracted` skip vs. dangling-dependent warning skip

Both behaviors live inside propagation (triggered by `retract`, `assert_node`, `challenge`, `add_justification`, etc.), and both are *skips* — but they represent fundamentally different situations and therefore have opposite observability contracts.

### What each behavior is

**Silent `_retracted` skip**
- `_retracted` is a boolean stored in `node.metadata` (not a real symbol; `find_retracted_symbol` found zero definitions).
- It is set intentionally during importer flows — `reasons/import_agent.py:311, 411–412, 467–468, 498` and `reasons/network.py:163` — whenever an imported belief is OUT/STALE in its source. It is cleared when the remote reports IN (`import_agent.py:425–426, 485`), and in `api.py:1637` it is popped from metadata.
- The comments at `import_agent.py:464–467` explain the purpose: "Set `_retracted` directly instead of using `retract_after`, … `_retracted` to keep OUT-with-justifications nodes locked." In other words, the flag encodes a *known, deliberate domain state*, not an error.
- The skip is silent per node: there is no `log` entry written when propagation encounters a `_retracted` node. The only visibility is aggregate, surfaced in the importer result dict (`api.py:1896` → `"claims_retracted"`, `import_agent.py:330, 511` → `"beliefs_retracted"`) and printed by the CLI (`cli.py:911–912, 942–943, 963`).

**Warning-and-skip for dangling dependents**
- A dependent is dangling when `node.dependents` contains an ID that is not in `net.nodes`. That violates a `Network` invariant — `tests/test_dependents_integrity.py:TestVerifyDependents` treats it as corruption.
- The behavior is implemented in `reasons/network.py:942`: `self._log("warn", dep_id, f"dangling dependent of {current_id}")`. The dependent is skipped, but a warning is *always* written.
- The full contract is pinned down in `tests/test_dangling_dependents.py`: the log entry has `action == "warn"`, `target == <ghost_id>`, `value` contains the word "dangling" and the *parent* node ID, and it carries a `timestamp` (`test_single_warning_logged`, `test_warning_includes_parent_node_id`, `test_warning_has_timestamp`). Multiple ghosts produce multiple warnings (`test_multiple_dangling_produce_multiple_warnings`), and the ghost ID must not enter the `visited` set so a later-real node of the same name still propagates (`test_dangling_does_not_enter_visited`).

### Why the asymmetry

- **`_retracted` = expected state.** An importer may push thousands of OUT claims; logging a warning per node would be noise and would surface correct behavior as a defect. The evidence is that there is *no* warning-hop test counterpart under `tests/` for `_retracted` — only the aggregate counters are tested.
- **Dangling = invariant violation.** Skipping a `KeyError`-driven crash is a robustness guard (issue #22, per the module docstring in `tests/test_dangling_dependents.py`), so the code chooses *skip + warn* so the corruption is diagnosable rather than silently swallowed. `test_dangling_dependents.py` and `tests/test_network.py:454` both assert the warning must be present.

### Operational implications

- If a belief is unexpectedly OUT with no warning, check `node.metadata.get("_retracted")`. It may simply be a stale import that hasn't been re-asserted by the source. Clearing is explicit (`import_agent.py:425–426, 485`; `api.py:1637`).
- If you see a `"warn"` entry with `value` starting "dangling dependent of…", treat it as a bug in the reverse-index (`dependents` set drift), which `verify_dependents()` / `_rebuild_dependents()` (`tests/test_dependents_integrity.py`) exist to detect and repair.
- Dangling IDs are *not* added to `visited`, so a "ghost" that later becomes a real node will still propagate. `_retracted` nodes are still nodes; they participate in propagation but are expected to stay OUT unless the source flips to IN.

### Observation gaps

The gathered data does not include the body of `Network._propagate`, so the exact `if node.metadata.get("_retracted"): continue` line (or equivalent) is inferred from the surrounding checks at `import_agent.py:411, 425, 477, 485` and the comment at `import_agent.py:402–404` ("`_retracted` is set so both `_propagate()` and remote IN flip correctly"). Additionally, `find_warning_skip_behavior` returned **0 matches** even though `network.py:942` literally contains `"warn"` — that search result is inconsistent and should be re-run before relying on it. To fully verify the "silent" claim about `_retracted`, inspect `reasons/network.py` around the `_propagate` loop for any `_log` call guarded by `_retracted`.

## Topics to Explore

- [function] `reasons/network.py:_propagate` — The single function where the `_retracted` skip and the dangling warning at line 942 both live; read it to confirm the "silent vs. warn" split.
- [file] `reasons/import_agent.py` — Source of almost every `_retracted` mutation; clarifies which states are considered "expected" vs. "corrupt".
- [function] `reasons/network.py:_log` — Defines the log-entry shape (`action`, `target`, `value`, `timestamp`) that tests and CLI consumers depend on.
- [file] `tests/test_dangling_dependents.py` — The authoritative spec for dangling-dependent semantics, including the "not in `visited`" rule.
- [general] `dependents-integrity` — How `verify_dependents()` / `_rebuild_dependents()` detect and repair the drift that produces dangling references in the first place.

## Beliefs

- `retracted-flag-is-silent` — Skipping a node with `_retracted` in `metadata` emits no per-node log entry; visibility is only via aggregate `claims_retracted` / `beliefs_retracted` counters.
- `dangling-dependents-warn` — `_propagate` emits a `"warn"` log entry with `target=<ghost_id>` and `"dangling"` in `value` when a `dependents` entry is missing from `net.nodes` (`reasons/network.py:942`).
- `retracted-flag-cleared-on-in` — Remote IN clears `_retracted` (`import_agent.py:425–426, 485`), enabling the node to propagate as IN again.
- `retracted-locks-out-with-justifications` — `_retracted` is set directly rather than via `retract_after` specifically to keep OUT-with-justifications nodes locked (`import_agent.py:464–467`).
- `dangling-not-in-visited` — Dangling dependent IDs are not added to `visited`, so a node added later under that ID still propagates correctly (pinned by `test_dangling_does_not_enter_visited`).
- `dangling-is-invariant-violation` — A dangling dependent is treated as corruption by `verify_dependents()` (`tests/test_dependents_integrity.py`), which is why the skip is warned rather than silent.
