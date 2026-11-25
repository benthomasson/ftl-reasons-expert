---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Determine whether phrases like `No topics found.` and `N topics from M nodes` are stable user-facing contracts or incidental output.

## Assessment

Based on the provided observations, these phrases are **de facto CLI-output contracts, but not documented public contracts**.

- `No topics found.` is emitted directly by the CLI at `reasons/cli.py:2313`, and the test at `tests/test_topics.py:140` asserts `"No topics found"` appears in stdout.
- `N topics from M nodes` is emitted at `reasons/cli.py:2317`, using the structured result from `api.topics()`:
  ```python
  print(f"\n{len(result['topics'])} topics from {result['total_nodes']} nodes")
  ```
  Tests lock parts of this wording:
  - `tests/test_topics.py:117` asserts `"topics from"` in stdout.
  - `tests/test_topics.py:134` asserts `"3 topics from"` in stdout.

That means changing these strings is not purely incidental: the test suite will catch some changes. However, the tests only assert **substrings**, not the full exact sentence, and `No topics found.` is tested without the trailing period. So the precise punctuation and complete phrasing are not fully locked.

There is also no documentation evidence that these phrases are an official public contract. `check_docs_for_phrases` found zero matches for `No topics found|topics from`.

The more stable contract is the structured result underneath:

- `reasons/api.py:3538` defines `topics(...)`.
- `reasons/cli.py:2317` consumes `result["topics"]` and `result["total_nodes"]`.
- `tests/test_topics.py` includes `TestTopicsApi` assertions on `result["topics"]` and `result["total_nodes"]`.
- `tests/test_topics.py:test_json_output` verifies that `--json` output is valid JSON and contains `"topics"` and `"total_nodes"`.

So the safest interpretation is:

- Treat the CLI phrases as **regression-protected human-readable presentation**.
- Do not treat the exact human-readable stdout as the primary machine contract.
- For automation, use `--json` or the `api.topics()` result structure.
- If intentionally changing the phrases, update `tests/test_topics.py` and consider documenting/changelogging if users may parse CLI text.

One caveat: the `locate_topics_command_in_cli` observation returned zero matches, which conflicts with the direct matches at `reasons/cli.py:2313` and `reasons/cli.py:2317`. That observation appears unreliable, so the direct source and test matches above are stronger evidence.

## Topics to Explore

- [file] `reasons/cli.py` — Inspect the `topics` command handler around lines 2313–2317 to see how text output and `--json` output paths are separated.
- [function] `reasons/api.py:topics` — Confirm the exact return shape, parameters, and error behavior that the CLI is formatting.
- [function] `reasons/mcp_server.py:topics` — Check whether the MCP tool exposes similar text or structured output and whether it shares the same contract.
- [file] `tests/test_topics.py` — Review each output assertion to understand exactly how much of the CLI wording is locked by tests.
- [repo] `reasons` — Search README, docs, and CHANGELOG for broader CLI-output promises, since the provided exact-phrase docs search found no matches.

## Beliefs

- `topics-cli-output-is-tested` — `tests/test_topics.py` asserts substrings `"No topics found"`, `"topics from"`, and `"3 topics from"`, so changing those CLI strings can break the test suite.
- `topics-api-contract-is-structured` — `reasons/api.py:topics` returns data consumed through `result["topics"]` and `result["total_nodes"]` in `reasons/cli.py:2317`, and API tests assert those fields.
- `topics-json-output-has-topics-total-nodes-keys` — `tests/test_topics.py:test_json_output` parses `--json` stdout and asserts that both `"topics"` and `"total_nodes"` keys exist.
- `topics-phrases-are-undocumented` — The provided docs search for `No topics found|topics from` returned zero matches, so these exact phrases are not documented as public contracts in the observed material.
- `topics-human-output-is-presentation` — The CLI phrases summarize values from the structured `api.topics()` result rather than defining separate data semantics.
