# Code Review Report

**Branch:** benthomasson/ftl-reasons#34
**Models:** claude, gemini
**Gate:** [CONCERN] CONCERN

## Disagreements

The following changes have different verdicts across models:

- **tests/test_nogood_id.py** [LOW]: claude: PASS, gemini: CONCERN

## claude [PASS]

### reasons_lib/network.py:Network.__init__
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Adds `_next_nogood_id: int = 1` to `__init__`. Clean, non-breaking addition. The initial value of 1 is correct for a fresh network with no nogoods. Tested in `test_init_sets_counter_to_one`.

### reasons_lib/network.py:Network._compute_next_nogood_id
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Scans all existing nogoods, extracts numeric IDs via `re.fullmatch(r"nogood-(\d+)", ng.id)`, finds the max, and sets `_next_nogood_id = max + 1`. Key design choices are sound: (1) `fullmatch` correctly ignores prefixed IDs like `agent:nogood-099`. (2) `\d+` handles any digit count, not just 3. (3) Defaults to `max_id = 0` so empty lists yield `_next_nogood_id = 1`. (4) Non-canonical IDs (e.g., `nogood-abc`, `custom-id`) are gracefully skipped. The approach of deriving-on-load rather than persisting the counter is the right call — it avoids schema changes and is inherently self-healing.

### reasons_lib/network.py:Network.add_nogood
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The core fix — replaces `len(self.nogoods) + 1` with `self._next_nogood_id` and increments after use. The counter is incremented unconditionally, which is correct — even if the nogood is later deleted, the ID space is consumed. The `{:03d}` formatting is preserved for backward compatibility but naturally overflows to 4+ digits for IDs >= 1000 (tested).

### reasons_lib/storage.py:Storage.load
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Calls `_compute_next_nogood_id()` immediately after loading all nogoods from SQLite, before loading repos and log entries. Placement is correct — counter must be set before any subsequent `add_nogood` calls on the loaded network. Tested in `TestNogoodIdStorageRoundTrip` and `TestStorageRoundTripWithDeletion`.

### reasons_lib/api.py:import_json
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Calls `_compute_next_nogood_id()` after the nogoods import loop. Placement is correct. Tested in `TestNogoodIdJsonImport` and `TestJsonImportWithGaps`.

### reasons_lib/import_beliefs.py:import_into_network
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Calls `_compute_next_nogood_id()` after the nogoods import block, inside the `if nogoods_text:` guard. This means it's only called when nogoods are actually imported, which is correct — the counter defaults to 1 from `__init__` and doesn't need recomputation if no nogoods were loaded. Tested in `TestNogoodIdBeliefImport`.

### tests/test_nogood_id.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Comprehensive test suite with 369 lines covering: core counter behavior (empty, single, gaps, prefixed, large numbers), the original deletion-collision bug, SQLite round-trips, JSON import, markdown import, and formatting edge cases. Test structure is well-organized into logical classes. One minor observation: `test_delete_last_then_add` verifies that after deleting `nogood-002`, adding a new one produces `nogood-002` (since max remaining is 001, next=2) — this means ID reuse is possible after deletion of the *latest* ID. This is technically correct per the derive-on-load design but worth noting: IDs are unique at any point in time but not historically unique. The issue only asked for "IDs remain unique regardless of deletions" (i.e., no collision with *existing* entries), which is satisfied.

### tests/test_nogood_id_qa.py (not in diff but present as untracked)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Additional QA test file (353 lines) covering end-to-end regression of the original bug, API-level round-trips with DB deletions, interleaved save/delete/load sequences across multiple cycles, backtracking interaction, idempotency of `_compute_next_nogood_id`, and non-canonical ID handling. This is thorough supplementary coverage. Note: there is some overlap with `test_nogood_id.py` (both test counter basics, prefixed IDs, etc.), but the QA file focuses on more realistic multi-cycle scenarios.

### Self-Review
**Limitations:** Could not run the test suite to confirm all tests pass. Could not verify whether existing tests in test_network.py, test_api.py, test_storage.py, test_import_beliefs.py, or test_backtracking.py break due to these changes (though the changes are additive/backward-compatible so breakage is unlikely). Did not verify the `Nogood` dataclass definition to confirm field names match usage.

### Feature Requests
- Include full function bodies for all modified functions in an "observations" section, not just the diff hunks, to verify surrounding logic
- Show callers of modified methods (e.g., all `add_nogood` callers) to verify integration completeness
- Auto-run the test suite and include results in the review context

## gemini [CONCERN]

### reasons_lib/network.py
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The implementation prevents collisions with *existing* entries by dynamically calculating the max ID rather than using `len()`. However, it does not persist a counter as suggested in the issue. Because the ID is derived dynamically, deleting the highest-numbered nogood(s) and reloading the network will cause those deleted IDs to be reused. This solves the immediate crash/collision bug but fails to guarantee that IDs remain globally unique regardless of deletions over time.

### reasons_lib/api.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Correctly integrates the counter update by calling `_compute_next_nogood_id()` after importing nogoods from JSON.

### reasons_lib/import_beliefs.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Correctly updates the ID counter by calling `_compute_next_nogood_id()` after importing nogoods from Markdown.

### reasons_lib/storage.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Successfully invokes `_compute_next_nogood_id()` after loading nogoods from SQLite. This ensures the runtime counter is correct based on the loaded database state.

### tests/test_nogood_id.py
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The tests are extremely thorough and cover the implemented logic well. However, because the implementation only partially addresses the issue, tests like `test_delete_last_then_add` and `test_delete_all_then_add` explicitly verify that IDs *are* reused when the most recent nogoods are deleted. This codifies the partial fix (reusing IDs) rather than enforcing strictly unique IDs across all deletions as requested.

### Self-Review
**Limitations:** None. The issue description, code changes, and tests provided sufficient context to evaluate the implemented approach against the reported problem.
