# Code Review Report

**Branch:** benthomasson/ftl-reasons#14
**Models:** claude, gemini
**Gate:** [CONCERN] CONCERN

## claude [CONCERN]

### `reasons_lib/import_beliefs.py:parse_beliefs`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Adds `"unless": []` default to the claim dict and parses `- Unless:` lines symmetrically with `- Depends on:`. The parsing logic mirrors the existing pattern exactly. `export_markdown.py:69` already emits `- Unless:`, so round-trip is supported. Tests `test_parses_unless` and `test_parses_unless_empty_default` cover both the populated and default cases.

### `reasons_lib/import_beliefs.py:import_into_network`
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Correctly builds `unless_in_network` filtered to claims that exist in the import set, then creates a `Justification` with both `antecedents` and `outlist`. The condition `if deps_in_network or unless_in_network` ensures a justification is created even when a node has only outlist references (no depends_on), which is the correct behavior for unless-only gating. Existing tests in `test_import_beliefs.py` and `test_import_json.py` validate the propagation behavior.

### `reasons_lib/import_agent.py:import_agent` (outlist addition)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The outlist remapping follows the same pattern as antecedent remapping — filter to `claim_by_id`, prefix with agent namespace. Moving the `Nogood` import to module-level is a clean improvement. Tests `test_import_agent_preserves_outlist` and `test_import_agent_supersession_preserved` directly validate the namespaced outlist behavior and propagation semantics.

### `reasons_lib/import_agent.py:import_agent_json`
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The function is well-structured and handles topological sorting, justification rebuilding, nogood import, and propagation correctly. However, I have two concerns:

1. **Antecedent validation asymmetry** (line 262 vs 263): Antecedents are namespaced unconditionally (`f"{prefix}{a}"` for all `a`), but outlist entries are filtered (`if o in nodes`). This means a justification could reference a non-existent antecedent node (e.g., an antecedent referencing a node not in the JSON). The markdown `import_agent` filters both antecedents and outlist to `claim_by_id`. This inconsistency could cause issues if the JSON export contains dangling antecedent references — though `_justification_valid` in the network handles missing nodes gracefully (treats missing antecedent as OUT, making justification invalid), so this is safe in practice but inconsistent.

2. **`nogoods_file` parameter silently ignored** (api.py line 626-639): When the JSON path is taken, `nogoods_file` is never read. JSON exports embed nogoods inline, so this is likely intentional, but a user passing `--nogoods` with a `.json` file gets no warning that the flag was ignored. Minor UX issue.

### `reasons_lib/api.py:import_agent` (JSON dispatch)
**Verdict:** CONCERN
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

File extension dispatch is clean. The JSON path correctly uses `_with_network` for transactional safety. However, the CLI help text at `cli.py:928` still says `"Path to the agent's beliefs.md"` — it should mention JSON is also accepted. The `nogoods_file` parameter is accepted but silently ignored for JSON imports (see concern above).

### `tests/test_import_agent.py` (new tests)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Four new test cases cover the critical scenarios well: (1) outlist preserved through markdown import with correct propagation (blocker IN → gated OUT), (2) supersession via outlist, (3) JSON import with full justification structure, (4) JSON import where outlist blocks. Tests verify both structural fidelity (outlist contents) and semantic correctness (truth values after propagation). The test for JSON outlist blocking (`test_import_agent_json_outlist_blocks`) is particularly valuable as it catches the common failure mode where propagation is skipped.

### `tests/test_import_beliefs.py` (new tests)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

`test_parses_unless` validates multi-value parsing. `test_parses_unless_empty_default` ensures backward compatibility with existing beliefs that don't have an `- Unless:` line.

### Self-Review
**Limitations:** Could not verify the full `_justification_valid` and `_compute_truth` logic to confirm that dangling antecedent references in `import_agent_json` are truly safe (the agent research reported it handles missing nodes, but I didn't read those lines directly). Did not run the test suite to confirm all 211+ tests pass with these changes. Could not verify the export side (`export` command) produces JSON with `outlist` keys that match what `import_agent_json` expects.

### Feature Requests
- Include the full body of modified functions in the diff context (not just the changed hunks), especially when hunks touch the middle of a function
- Show callers of modified functions to verify nothing else passes data into them with assumptions about the old signature
- When CLI argument parsers are wired to API functions, include the argparse definitions alongside the API changes

## gemini [PASS]

### reasons_lib/api.py:import_agent
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** MEETS
**Test Coverage:** COVERED
**Integration:** WIRED

The function cleanly branches to handle JSON files by dispatching to `import_agent_json` while preserving existing Markdown import logic. The file reading and context manager use are correct.

### reasons_lib/import_agent.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** MEETS
**Test Coverage:** COVERED
**Integration:** WIRED

The modifications to `import_agent` correctly extract and namespace `unless` relationships to build the `outlist`. The new `import_agent_json` function accurately reconstructs the belief network from a JSON export, correctly applying topological sorting to handle dependencies gracefully and appropriately restoring outlist constraints to preserve non-monotonic relationships.

### reasons_lib/import_beliefs.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** MEETS
**Test Coverage:** COVERED
**Integration:** WIRED

`parse_beliefs` is properly updated to recognize `- Unless: ` directives and parse them into a list, matching the existing `Depends on` pattern. `import_into_network` maps these parsed values effectively into `outlist` when generating justifications.

### tests/test_import_agent.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** MEETS
**Test Coverage:** COVERED
**Integration:** WIRED

Good test coverage. Includes tests for markdown `unless` import, JSON export import, supersession cases via outlists, and verification that truth values appropriately propagate when outlist nodes become IN.

### tests/test_import_beliefs.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** MEETS
**Test Coverage:** COVERED
**Integration:** WIRED

Includes specific parser tests verifying that `- Unless: ` strings correctly parse into lists and that empty defaults are properly handled.

### Self-Review
**Limitations:** None. The diff contained complete test coverage and full contexts for the critical methods being updated. All logical paths for the new JSON import handling and outlist properties could be verified.
