# Code Review Report

**Branch:** benthomasson/ftl-reasons#35
**Models:** claude, gemini
**Gate:** [BLOCK] BLOCK

## Disagreements

The following changes have different verdicts across models:

- **tests/test_nogood_id.py** [LOW]: claude: PASS, gemini: CONCERN

## claude [CONCERN]

### reasons_lib/network.py:Network.__init__
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The `_next_nogood_id: int = 1` counter is initialized correctly. It starts at 1, which matches the old `len(self.nogoods) + 1` behavior for fresh networks. No observable difference for callers unless they delete nogoods.

### reasons_lib/network.py:Network.add_nogood
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Core fix is minimal and correct. The counter is incremented unconditionally after generating the ID, so even if `add_nogood` partially fails after this point (e.g., during backtracking), the ID is still consumed — this is the right choice since partial failures shouldn't reclaim IDs. The `:03d` formatting is preserved from the original code.

### reasons_lib/storage.py:Storage.save
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The `network_meta` table is cleared and rewritten during each save, consistent with the existing "clear and rewrite" strategy used for all other tables. The counter is persisted as a string in a key-value store, which is simple and adequate.

### reasons_lib/storage.py:Storage.load (metadata section)
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The backward compatibility logic is functionally correct but the `except Exception: pass` at line 210 is overly broad. If the `network_meta` table exists but the query fails for a different reason (e.g., disk corruption, `int()` conversion failure on a malformed value), the error is silently swallowed and the counter defaults to 1. This could cause the exact ID collision the fix is meant to prevent. A narrower catch (e.g., `except sqlite3.OperationalError`) would be safer — that's the specific error when a table doesn't exist. The `int(row[0])` conversion could also raise `ValueError` on bad data; that should probably propagate rather than being silently ignored. **However**, this pattern matches the existing `except Exception: pass` used for repos loading at line 219, so it's at least consistent with the existing codebase style.

### reasons_lib/storage.py:SCHEMA (network_meta table)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The `CREATE TABLE IF NOT EXISTS network_meta` is idempotent (uses `IF NOT EXISTS`), so it's safe for existing databases — they'll get the table created on first `Storage()` instantiation. This means the "old database without meta table" backward compat path in `load()` will only be hit if an old database was never opened by new code before loading. The schema is simple and appropriate for a key-value metadata store.

### reasons_lib/api.py:import_json (nogood counter update)
**Verdict:** CONCERN
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Logic is correct — uses `max()` so importing multiple nogoods with non-sequential IDs sets the counter to max+1. The `import re` is placed inside the function body rather than at module top. This is inconsistent with `import_beliefs.py` which imports `re` at the top level, but matches the existing pattern in this function (the `from . import Nogood` on line 926 was already a local import). Minor style inconsistency, not a correctness issue.

### reasons_lib/import_beliefs.py:import_into_network (nogood counter update)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Same `max()` logic as `api.py`. Uses `re` from the module-level import. Correctly placed inside the loop so each imported nogood updates the counter.

### tests/test_nogood_id.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Comprehensive test coverage with 9 tests across 4 classes. Tests cover: basic incrementing, delete-last, delete-all, save/load round-trip, delete+persist, old DB without meta table, old DB with nogoods (counter derivation), JSON import, and beliefs import. The backward compat tests are particularly well-designed — `test_old_db_without_meta_table` drops the table and `test_old_db_with_nogoods_derives_counter` deletes only the meta row. Both are realistic migration scenarios. One note: `test_old_db_without_meta_table` (line 100) won't actually test the `except Exception` path because `_init_schema()` in the new `Storage()` constructor will re-create the `network_meta` table. The test still works because the table will be empty (row was never inserted), hitting the `elif network.nogoods` branch — but since there are no nogoods, it falls through to the default `_next_nogood_id = 1`. This is fine because it tests the "empty meta table" case rather than the "missing meta table" case, but the docstring claims it tests "Databases created before this fix should still load" which is slightly misleading.

### Self-Review
**Limitations:** Could not run the test suite to verify all tests pass. Could not verify whether any existing tests in `test_storage.py` or `test_backtracking.py` exercise `add_nogood` and might be affected by the ID format change (they shouldn't be, since the format `nogood-NNN` is unchanged, only the counter source changed).

### Feature Requests
- Include existing test files that test modified functions (e.g., `test_storage.py`, `test_api.py`) to verify no existing assertions break
- Show callers of modified functions to verify all paths that create nogoods are updated

## gemini [BLOCK]

### reasons_lib/api.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Safely handles updating the counter when importing nogoods from JSON. It correctly derives the max ID without resetting the counter downwards.

### reasons_lib/import_beliefs.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Correctly ensures the monotonic counter is updated when importing nogoods from a markdown file.

### reasons_lib/network.py
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Correctly implements the monotonic counter `_next_nogood_id` and securely increments it, ensuring that IDs are never reused even after deletions.

### reasons_lib/storage.py
**Verdict:** BLOCK
**Correctness:** BROKEN
**Spec Compliance:** N/A
**Test Coverage:** PARTIAL
**Integration:** WIRED

The backward-compatibility logic for old databases in `load()` is structurally broken. If the `network_meta` table genuinely does not exist, `self.conn.execute` raises an exception and execution immediately jumps to `except Exception: pass`. This entirely skips the `elif network.nogoods:` block meant to derive the max ID, silently leaving the counter at 1 and causing ID collisions. 

Additionally, `save()` will crash on an old database if the table is missing because it calls `self.conn.execute("DELETE FROM network_meta")` outside any try/except block. If `Storage.__init__` guarantees the table is always created, the exception logic in `load()` is misleading and hides actual DB errors. The derivation logic must execute reliably if the persisted value is missing, regardless of whether the query failed or returned no rows.

### tests/test_nogood_id.py
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** PARTIAL
**Integration:** WIRED

The tests mask the missing table bug in `storage.py`. `test_old_db_without_meta_table` does not add any nogoods before saving, so it expects an ID of 1 whether derivation ran or not. Furthermore, `test_old_db_with_nogoods_derives_counter` uses `DELETE FROM network_meta` instead of dropping the table. This means it only tests the path where the table exists but is empty, entirely missing the exception branch where derivation is skipped.

### Self-Review
**Limitations:** I could not see the complete implementation of `Storage.__init__` to confirm if the sqlite schema is strictly initialized on connection or only executed when `init_db()` is explicitly called. However, the inconsistent behavior between `save()` (crashing if no table) and `load()` (swallowing the exception and skipping derivation) stands as a definitive issue either way.

### Feature Requests
- Provide the full source context for modified classes (e.g., constructors) instead of only modified functions, so lifecycle behaviors like database migrations/schema execution can be accurately verified.
