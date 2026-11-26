---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: Where `_assign_topics` is invoked and whether callers pass lists or generators.

## Where `_assign_topics` is invoked

`_assign_topics` is defined once in `reasons/build_wiki.py:24`:

```python
def _assign_topics(node_ids, topics):
```

It is invoked from five observed call sites:

| Location | Call |
|---|---|
| `reasons/api.py:3622` | `groups = _assign_topics(node_ids, topics_result["topics"])` |
| `tests/test_build_wiki.py:52` | `groups = _assign_topics(node_ids, topics)` |
| `tests/test_build_wiki.py:59` | `groups = _assign_topics(node_ids, topics)` |
| `tests/test_build_wiki.py:64` | `groups = _assign_topics([], topics)` |
| `tests/test_build_wiki.py:68` | `groups = _assign_topics(["some-node"], [])` |

The other references are imports or the definition itself:

- `reasons/api.py:3580` imports it: `from .build_wiki import _assign_topics, build_wiki as _build_wiki`
- `tests/test_build_wiki.py:12` imports it for tests
- `reasons/build_wiki.py:24` is the function definition

So the production call is in `reasons/api.py:3622`, and the test calls are in `tests/test_build_wiki.py`.

## Do callers pass lists or generators?

The observations show clear list usage only at the two test call sites with literal arguments:

- `tests/test_build_wiki.py:64` passes `[]` as `node_ids`
- `tests/test_build_wiki.py:68` passes `["some-node"]` as `node_ids` and `[]` as `topics`

No observed call site contains a generator expression such as `(x for x in ...)` or a `yield`.

However, the remaining call sites pass variables:

- `reasons/api.py:3622` passes `node_ids` and `topics_result["topics"]`
- `tests/test_build_wiki.py:52` and `:59` pass `node_ids` and `topics`

The provided observations do not show the definitions or assignments of those variables, so their exact runtime types are not proven here. Based on the `api.topics` tests in `tests/test_topics.py`, `topics_result["topics"]` is likely a list, since `api.topics(...)` returns a dict with a `"topics"` key whose value is checked against `[]` and indexed as a list. But the observation does not show the immediate construction of `topics_result` at `reasons/api.py:3622`.

Net: the evidence supports list arguments, and there is no observed generator argument. To be certain, inspect the surrounding code that builds `node_ids` and `topics_result["topics"]` before `reasons/api.py:3622`, and read `tests/test_build_wiki.py` around lines 45–70.

## Topics to Explore

- [file] `reasons/build_wiki.py` — Read the full `_assign_topics` implementation to see whether it iterates arguments once or multiple times, which determines whether generators are safe.
- [file] `reasons/api.py` — Inspect the code before line 3622 to see how `node_ids` and `topics_result["topics"]` are constructed.
- [file] `tests/test_build_wiki.py` — Read lines around 45–70 to see the concrete values assigned to `node_ids` and `topics` in the variable-based test calls.
- [function] `reasons/api.py:topics` — Confirm the return shape of the topics API and whether `result["topics"]` is guaranteed to be a list.
- [general] `generator-argument-compatibility` — Check whether any internal or future caller passes a generator into `_assign_topics` and whether repeated iteration would break.

## Beliefs

- `assign-topics-has-five-observed-call-sites` — `_assign_topics` is invoked from `reasons/api.py:3622` and `tests/test_build_wiki.py` lines 52, 59, 64, and 68.
- `assign-topics-definition-has-no-type-annotations` — The function is defined as `def _assign_topics(node_ids, topics):` at `reasons/build_wiki.py:24`, with no annotations constraining either argument to a list or generator.
- `explicit-test-callers-pass-list-literals` — Tests at `tests/test_build_wiki.py:64` and `:68` pass list literals such as `[]` and `["some-node"]`.
- `api-imports-assign-topics` — `reasons/api.py:3580` imports `_assign_topics` from `.build_wiki`.
- `no-generator-arguments-are-observed` — The gathered call-site text contains no generator expression or `yield` passed to `_assign_topics`, though some callers pass variables whose exact types are not shown.
