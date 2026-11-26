---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Search for tests that assert publish ordering relative to save/commit; the provided search found no such tests.

## Summary

The claim **"the search found no tests asserting publish ordering relative to save/commit"** is technically what the search returned, but the observations strongly suggest the search itself was incomplete — not that such tests are absent. Several searches returned zero matches for tokens that are demonstrably present in the returned file contents, so the "no such tests" conclusion should be treated as unverified rather than confirmed.

## What the searches actually did (and where they went wrong)

1. **`find_publish_tests` (pattern `publish`) → 0 matches.**
   This is the biggest red flag. `tests/test_pubsub.py` plainly contains `publish`, `publish(`, and `def test_publish_empty_list_skipped` (visible in `read_pubsub_tests`). A zero-match result means the search scope or pattern handling did not cover `tests/test_pubsub.py`. The same applies to `find_commit_save_tests` (pattern `commit|save`) → 0, despite `tests/test_pin_sources.py:70` containing `commit` — the pipeline may be treating `|` literally rather than as alternation.

2. **`find_publish_call_sites` (pattern `publish\(`) → 0 matches.**
   Again contradicted by the visible test file, which calls `publish(events, "db1.db")`, `publish([], "t.db")`, etc. So call-site discovery is unreliable here.

3. **`locate_storage_save_commit` → 0 matches.**
   No source for `_with_network`, `save`, or `commit` was returned at all. The pubsub docstring references `_with_network.__exit__` as the caller (lines 1–6 and the `publish` docstring at `reasons/pubsub.py:70`), but that function is not in the observations. Without it, the ordering claim cannot be checked.

4. **`find_ordering_assertions` → 1 match, and it is not relevant.**
   The single hit is `tests/test_pin_sources.py:70`:
   ```python
   def test_sha_changes_after_commit(self, git_repo):
   ```
   This is about a **git commit** changing a SHA for pinned sources — unrelated to database save/commit ordering versus event publication.

## What we *can* say about ordering from the code shown

The ordering behavior is asserted **only in documentation**, not in the visible tests:

- Module docstring (`reasons/pubsub.py`, top of file): *"Subscribers receive batched events after each `_with_network` write transaction completes (after save)."*
- `publish` docstring (`reasons/pubsub.py:70`): *"Called by `_with_network.__exit__` after save completes."*

The visible integration tests in `tests/test_pubsub.py` (`TestApiIntegration`, e.g. `test_add_node_emits_event`, `test_retract_cascade_emits_events`, `test_assert_emits_events`) only verify:

- that events are delivered after an API call returns,
- that they contain the expected `node_id`, `old_truth_value`, and `new_truth_value`.

They do **not** verify that a DB commit happened *before* the subscriber was invoked — for example, no test asserts that a subscriber reading the DB inside its callback sees the committed row. So even the visible tests do not constitute an ordering assertion.

There is also a coverage gap: `tests/test_pubsub.py` has **347 lines**, and only **lines 0–200** were returned. Any ordering-related test could plausibly live in the missing 147 lines.

## Conclusion

The "no tests found" result is best interpreted as **a search tooling gap**, not a definitive absence of coverage. What the observations support is narrower: **no ordering test is visible in the returned slice**, and the only ordering claim in code is a docstring in `reasons/pubsub.py`. To prove or disprove coverage you would need the `_with_network` implementation and the remainder of `tests/test_pubsub.py`.

## Topics to Explore

- [file] `reasons/pubsub.py` — Read the full module and docstrings; `publish` at line 70 is documented as post-save but the caller is not shown here
- [function] `reasons/pubsub.py:publish` — The dispatch point; confirm whether it can be invoked inside a transaction and what guarantees the `_with_network.__exit__` contract gives
- [file] `tests/test_pubsub.py` — Read lines 200–347 (not returned) to check for any ordering/commit assertions
- [general] `_with_network-transaction-hook` — Locate the implementation that wraps writes and calls `publish`; this is the only place the ordering invariant can be enforced or violated
- [repo] `ordering-test-search-config` — Investigate why `publish` and `publish\(` matched zero files despite obvious occurrences, to know when to trust future search results

## Beliefs

- `publish-payload-contract` — `publish(events, db_path)` returns `None`, skips dispatch when `events` is empty, and routes to both global (`db_path=None`) and db-scoped subscribers.
- `subscriber-exceptions-are-swallowed` — `publish` wraps each subscriber call in `try/except Exception` and logs via `logger.exception`, so misbehaving subscribers do not propagate errors to the write path (`reasons/pubsub.py`, lines ~84–90).
- `pubsub-docstring-claims-after-save-ordering` — The module docstring and `publish` docstring both state that events are emitted only after the `_with_network` write transaction completes/saves.
- `cascade-events-are-batched-to-final-state` — Per the module docstring, intermediate cascade states are not published; only the final stable truth-value state is visible in a publish batch.
- `visible-tests-do-not-verify-commit-before-publish` — The `TestApiIntegration` tests in `tests/test_pubsub.py` assert event contents after API calls return, but never assert that a commit precedes subscriber invocation.
- `ordering-search-results-are-unreliable` — `find_publish_tests` and `find_publish_call_sites` returned zero matches despite `tests/test_pubsub.py` containing `publish` identifiers and calls, indicating the search did not cover the relevant file.
