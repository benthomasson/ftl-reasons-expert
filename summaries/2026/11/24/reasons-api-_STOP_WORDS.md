---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Function: _STOP_WORDS in reasons/api.py

## `_STOP_WORDS` — Search Query Stop Word Set

### Purpose

`_STOP_WORDS` is a module-level `frozenset` of ~150 lowercase English stop words. It is **not an NLP artifact** — it exists purely to improve the precision of the two text-**search** functions in this module:

- `lookup()` — substring/AND matching over the full belief block
- `_fts_search()` — SQLite FTS5 query construction

Stop words like *"the"*, *"is"*, *"of"*, *"what"*, and *"which"* carry near-zero information for retrieval, so leaving them in a query forces every result to match noise tokens. The set includes a short curated tail of **project-specific filler** — `"specific"`, `"specifically"`, `"particular"`, `"particularly"`, `"included"`, `"including"`, `"within"` — which are the kind of words written in belief texts in this repo but rarely discriminative for search.

### Contract

`_STOP_WORDS` is a value with no behavior of its own; its contract is with its two consumers.

**Invariants:**
- Every element is a **single, lowercase alphanumeric token**. This matches the output of `re.findall(r'\w+', ...)`, which is the tokenizer used by both call sites.
- The object is a `frozenset`, so it is immutable and hashable; membership tests are O(1).
- Elements are all length > 1 (nothing like `"i"` or `"a"`? — `"a"` **is** present, but the callers also apportion a separate `len(t) > 1` guard, so `"a"` is filtered twice).

**Assumptions not enforced by the type system:**
1. **Lowercase input.** Callers must `.lower()` the token **before** testing membership. `"The"` is not in the set, only `"the"`.
2. **Single-token input.** `"in order to"` will never match a set entry; only `"in"`, `"order"`, and `"to"` are individually addressable after tokenizing.
3. **English.** Nothing here helps non-English text.
4. **Fallback is not an error.** When every token is filtered out, the callers deliberately *reuse the unfiltered terms* rather than returning "no results", silently disabling the stop-word filter as a graceful degradation.

### Parameters

`_STOP_WORDS` is a constant, not a function; it has no parameters. Its **elements** drive the behavior of two call sites:

```python
# In lookup()
query_terms = [t.lower() for t in raw_terms
               if t.lower() not in _STOP_WORDS and len(t) > 1]
if not query_terms:
    query_terms = [t.lower() for t in raw_terms if len(t) > 1]

# In _fts_search()
terms = [t for t in raw_terms if t.lower() not in _STOP_WORDS and len(t) > 1]
if not terms:
    terms = [t for t in raw_terms if len(t) > 1]
if not terms:
    return []
```

Two subtle differences worth noting:
- `lookup` **lowercases** the surviving terms (necessary for its substring matcher).
- `_fts_search` **preserves original case** for terms handed to FTS5, relying on FTS5's case-insensitive tokenizer/stemmer. Because the membership test already lowercases the LHS, stop-word filtering is still case-insensitive in both.

### Return Value

N/A — it's a constant. Callers return `query_terms` / `terms` **derived** from membership in this set.

### Algorithm (how the set is consumed)

1. `re.findall(r'\w+', query)` splits the raw query into alphanumeric tokens. Multi-word phrases, hyphens, apostrophes, and punctuation are **not** preserved.
2. Each token is lowercased and compared against `_STOP_WORDS` and against a `len(t) > 1` guard.
3. Survivors become the actual search terms.
4. If the survivor list is empty, the tokenizer output is re-derived **without** the stop-word filter (but still with the `len > 1` guard). This makes a fully-stop-word query like `"the of is"` still run a search for `["the", "of", "is"]`.
5. `_fts_search` additionally returns `[]` if even the fallback yields nothing (e.g. a query of only single characters like `"a b c"`).

### Side Effects

None. `frozenset` is immutable, so nothing anywhere in the module mutates `_STOP_WORDS`. Its construction is a one-time cost at import time.

### Error Handling

`_STOP_WORDS` cannot raise. Callers do not raise on fully filtered queries — they degrade to unfiltered search (step 4 above). The only explicit "empty" return is `_fts_search` returning `[]` for a query that tokenizes to nothing at all.

### Usage Patterns

- Called indirectly by any user of `lookup()` or `search()` (via `_fts_search`).
- Any addition or removal of an entry changes the result set of those two search APIs. Because the two consumers implement filtering *inline*, adding words here silently affects **both** — there is no single helper like `_tokenize(query)`. That is the main maintenance hazard.

### Dependencies

- **Internal:** none — `_STOP_WORDS` is a leaf value. It is imported/read from `lookup()` and `_fts_search()` in this same module. A separate, overlapping set `_TOPIC_STOP_WORDS` exists later in the file and is used by `topics()` and `build_wiki()`; it is *not* `_STOP_WORDS`.
- **External:** none.

### Common Pitfalls

- Confusing `_STOP_WORDS` (search) with `_TOPIC_STOP_WORDS` (topic extraction from node IDs). They are curated differently and are NOT interchangeable — `_TOPIC_STOP_WORDS` contains things like `"use"`, `"used"`, `"based"`, `"per"`, `"via"`, `"get"`, `"set"`, `"one"`, `"two"`, `"new"` that make sense only for tokenizing node IDs like `derived-auth-token-v2`. `_STOP_WORDS` includes things like `"specifically"`, `"including"`, `"within"`, `"upon"`-style discourse words.
- The set includes `"a"` but not `"I"`/`"am"`/`"were"` — wait, `"were"` *is* there but `"am"` is **missing**. This is a genuinely incomplete English stop list; it's tuned for query precision, not linguistic completeness. Don't "fix" it in isolation without considering the impact on `_fts_search` relaxation behavior.
- Adding a word here affects both `lookup` **and** `search`, and can subtly change relevance ranking because `_fts_search` uses the term list length to decide whether to invoke progressive relaxation (`if not results and len(terms) > 2`).

---

## Topics to Explore

- [function] `reasons/api.py:_fts_search` — How FTS5 queries are constructed and the `_MAX_RELAXATION_QUERIES` bucketing loop
- [function] `reasons/api.py:lookup` — The other consumer; also uses progressive relaxation by combination of terms
- [function] `reasons/api.py:_fts_query` — Where surviving terms are actually wrapped in `"..."` and passed to `nodes_fts MATCH`
- [general] `_TOPIC_STOP_WORDS-vs-_STOP_WORDS` — Why the module contains two stop word sets and how they diverge
- [file] `tests/test_api.py` — Any tests covering stop-word filtering behavior in `search` / `lookup`

## Beliefs

- `stop-words-filtered-by-lowercase` — Every entry in `_STOP_WORDS` is lowercase; callers must lowercase tokens before membership testing for case-insensitive filtering.
- `stop-words-are-single-token` — All members are single alphanumeric tokens, compatible with `re.findall(r'\w+', query)` output but never with phrases.
- `stop-words-immutable` — `_STOP_WORDS` is a `frozenset`, so no runtime code can mutate it; changes only happen at source level.
- `stop-words-duplicated-filtering-logic` — Both `lookup` and `_fts_search` independently implement the same stop-word filter followed by the same fallback to unfiltered terms when all are filtered.
- `stop-words-separate-from-topic-stop-words` — `_TOPIC_STOP_WORDS` is a distinct, differently curated set used only by `topics()` / `build_wiki()`, not by search functions.
- `stop-words-case-asymmetry` — `_fts_search` preserves the original case of surviving terms for FTS5, while `lookup` lowercases them before substring matching.
