---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Function: _assign_topics in reasons/build_wiki.py

## Purpose

`_assign_topics` groups belief/node IDs into topic buckets by splitting each node ID into segments and matching those segments against a set of topic labels. It is a private helper used to prepare a `{topic_label: [node_id, ...]}` mapping for later wiki generation; `build_wiki` consumes a mapping of this shape as its `groups` argument.

The docstring says “best-matching topic,” but the implementation is **first matching segment wins**, where segment order is determined by the node ID’s split order, not by topic-list order.

## Contract

### Preconditions

- `node_ids` is an iterable of strings.
- `topics` is a **re-iterable** collection of mapping-like objects, each containing a hashable `"topic"` key.
  - This is important because `topics` is iterated twice: once to build `topic_set`, once to build `groups`.
  - Passing a generator or other one-shot iterator can break the function.
- Topic labels are expected to be strings that can exactly match split ID segments.
- Matching is case-sensitive and exact; no lowercasing, stemming, or stop-word filtering is performed.

### Postconditions

- Every input node occurrence ends up in exactly one returned list.
- A node is placed in the first topic bucket whose label exactly matches one of its split ID segments.
- Nodes with no matching segment are placed under `"Other"`.
- Empty buckets are omitted from the returned dictionary.
- Within each bucket, node IDs preserve the iteration order of `node_ids`.
- The input `topics` and `node_ids` are not mutated, though iterators may be consumed.

### Invariants / assumptions not enforced by types

- `topics` yields the same logical topic labels on both iterations.
- Topic labels used in `topic_set` must also exist as keys in `groups`; otherwise `groups[word].append(...)` raises `KeyError`.
- A topic literally named `"Other"` collides with the fallback bucket.
- Topics of length `<= 2` can never match because split segments of length `<= 2` are filtered out.
- Topic labels containing separators such as `-`, `.`, `_`, or `:` will not match a single segment.

## Parameters

- `node_ids`: iterable of node ID strings, e.g. `["sl-justification-mechanism", "tms-node"]`.
  - If a generator is passed, it is consumed.
  - Non-string elements cause `re.split` to raise `TypeError`.
- `topics`: iterable of dictionaries/mappings, each like `{"topic": "justification"}`.
  - A missing `"topic"` key raises `KeyError`.
  - Duplicate topic labels collapse into one dictionary key.
  - If `"Other"` is supplied as a real topic, it shares the same bucket as the unmatched fallback.
  - If `topics` is a one-shot iterator, the second iteration used to build `groups` sees nothing; a later matched word then raises `KeyError` on `groups[word]`.

## Return Value

Returns a dictionary:

```python
{topic_label: [node_id, ...], ...}
```

Only non-empty groups are included. Therefore:

- Topics with no matching nodes are absent.
- `"Other"` is absent if every node matched.
- If `topics` is empty and `node_ids` is non-empty, the result is `{"Other": [...]}`.
- If both are empty, the result is `{}`.

Callers must not assume every topic key is present. Use `.get(label, [])` or membership checks when iterating expected topics.

## Algorithm

1. Build `topic_set` from all `t["topic"]` values:
   ```python
   topic_set = {t["topic"] for t in topics}
   ```
2. Build an empty list bucket for every declared topic:
   ```python
   groups = {t["topic"]: [] for t in topics}
   ```
3. Add the fallback bucket:
   ```python
   groups["Other"] = []
   ```
4. For each node ID:
   - Split it on any of the characters `-`, `.`, `_`, or `:` using `re.split(r'[-._:]', nid)`.
   - Discard empty segments and segments with length `<= 2`.
   - Iterate through the remaining segments in order.
   - If a segment exactly matches a topic label in `topic_set`, append the node ID to that topic bucket and stop checking further segments for this node.
   - If no segment matches, append the node ID to `groups["Other"]`.
5. Return only the non-empty buckets:
   ```python
   return {k: v for k, v in groups.items() if v}
   ```

Because matching stops at the first matching segment, a node like `"foo-bar-baz"` with topics `"bar"` and `"baz"` is assigned to `"bar"`, not `"baz"`.

## Side Effects

- No file I/O.
- No mutation of `topics` or `node_ids`.
- Consumes iterators passed as `node_ids` or `topics`.
- Creates and mutates only local dictionaries/lists.

## Error Handling

There is no explicit validation or exception handling. Possible exceptions include:

- `KeyError` if a topic mapping lacks `"topic"`.
- `KeyError` on `groups[word].append(...)` if `topics` is a one-shot iterator or otherwise yields inconsistent topics between the two comprehensions.
- `TypeError` if `topics` is not iterable, if a topic label is unhashable, if `node_ids` is not iterable, or if a node ID is not a string/bytes-like value accepted by `re.split`.
- `re.error` is not realistically expected because the regex is constant and valid.

Errors are not swallowed or converted.

## Usage Patterns

Typical usage is to prepare topic groups for `build_wiki`:

```python
groups = _assign_topics(node_ids, topics)
build_wiki(node_details, groups, output_dir)
```

Caller obligations:

- Pass `topics` as a list/tuple or other re-iterable collection, not a generator.
- Ensure topic labels are exact, case-sensitive segments that appear in node IDs after splitting.
- Handle absent buckets such as `"Other"` and empty topics.
- Sort node IDs later if deterministic wiki output is required; this function preserves input order but does not sort.

The shown `reasons/build_wiki.py` file defines `_assign_topics` but does not call it directly, so the call site is outside this excerpt or in tests.

## Dependencies

- Standard library `re`, imported at module level.
- No other internal helpers or external modules are used by this function.

## Topics to Explore

- [file] `tests/test_build_wiki.py` — Confirms expected grouping behavior and edge cases for `_assign_topics`.
- [function] `reasons/build_wiki.py:build_wiki` — Consumes the `{group_label: [node_id, ...]}` mapping produced here.
- [function] `reasons/build_wiki.py:_page_name` — Sanitizes topic labels into markdown filenames after grouping.
- [general] `topic-generation` — How topic dictionaries with `"topic"` keys are produced before assignment.
- [general] `assign-topics-call-sites` — Where `_assign_topics` is invoked and whether callers pass lists or generators.

## Beliefs

- `assign-topics-splits-on-four-delimiters` — Node IDs are split on hyphen, dot, underscore, and colon only.
- `assign-topics-ignores-short-segments` — Split ID segments of length two or less are discarded before matching.
- `assign-topics-first-match-wins` — A node is assigned to the first split segment that exactly matches a topic label.
- `assign-topics-omits-empty-groups` — The returned dictionary excludes topic and `"Other"` buckets whose lists are empty.
- `assign-topics-requires-topic-key` — Every element of `topics` must contain a `"topic"` key or `KeyError` is raised.
