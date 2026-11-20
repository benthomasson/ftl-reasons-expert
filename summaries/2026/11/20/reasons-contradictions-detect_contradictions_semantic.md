---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Function: detect_contradictions_semantic in reasons/contradictions.py

### Purpose

`detect_contradictions_semantic` is the semantic-clustering variant of contradiction detection. Instead of sending random batches of all `IN` beliefs to the LLM, it first groups beliefs by semantic similarity using `list_clusters`, then asks the LLM to find contradictory `NOGOOD` sets within each cluster.

It exists to improve recall for topically related beliefs: two beliefs that contradict each other are more likely to be placed in the same cluster and therefore seen together in the same LLM prompt. The tradeoff is that contradictions spanning different clusters may be missed.

### Contract

Preconditions, roughly:

- `nodes` is a mapping of `node_id -> node_data`.
- Each selected node is expected to be a dict-like object with `.get`.
- `truth_value == "IN"` marks beliefs eligible for checking.
- If `belief_ids` is `None`, node keys must be sortable because the function uses `sorted(nodes.items())`.
- `list_clusters` must return a dict containing a sized `"clusters"` collection.
- Each cluster must contain `"beliefs"` entries with an `"id"` key.
- `invoke_model`, `parse_contradiction_response`, and `_flush_results` must behave according to their existing contracts.

Postconditions:

- Returns a list of parsed contradiction dicts.
- Only beliefs marked `IN` at selection time are normally considered.
- Clusters with fewer than two belief IDs are skipped entirely.
- The LLM is called per cluster batch, not across all beliefs globally.
- If `output_path` is provided, parsed results are flushed incrementally.
- Failures in batch-level LLM parsing or flushing are converted into warnings; results from other batches are retained.
- No deduplication is performed across clusters or batches.

Important invariant: `parse_contradiction_response` receives `valid_ids=set(belief_ids)`, i.e. the global set of checked `IN` belief IDs, not just the current cluster or batch. The parser filters claims against that global set.

### Parameters

| Parameter | Meaning | Edge cases |
|---|---|---|
| `nodes` | Dict of `node_id -> node data` from `export_network()`. | Missing `truth_value` excludes the node. Missing `text` becomes `""`. Non-dict node values can raise `AttributeError`. |
| `belief_ids` | Optional explicit IDs to check. | If `None`, all `IN` nodes are selected, sorted by node ID. If provided, IDs not in `nodes` or not marked `IN` are silently dropped. Duplicates are not explicitly removed, but later dict/set usage deduplicates. |
| `model` | LLM model name passed to `invoke_model`. | Defaults to `"claude"`. Not validated by this function. |
| `timeout` | LLM timeout in seconds. | Passed directly to `invoke_model`. |
| `embedding_model` | Sentence-transformers model name for clustering. | If falsy/`None`, falls back to `DEFAULT_MODEL` imported from `.cluster`. |
| `output_path` | Optional path for incremental contradiction-plan writes. | If `None`, no writes. If set, results are written only when a batch produces at least one parsed contradiction. |

### Return Value

Returns a list of dicts shaped like:

```python
[
    {
        "id": "short-kebab-id",
        "claims": ["belief-id-1", "belief-id-2"],
        "analysis": "...",
        "severity": "High",
    }
]
```

The function can return `[]` when:

- no candidate `IN` beliefs remain after filtering,
- clustering produces no clusters with at least two beliefs,
- every batch fails,
- the LLM finds no contradictions,
- parsed results are filtered out because fewer than two claims remain.

The caller must handle partial results: some cluster batches may fail and be skipped while others succeed. The caller also must not assume completeness, deduplication, or cross-cluster coverage.

### Algorithm

1. Lazily import `list_clusters` and `DEFAULT_MODEL` from `.cluster`.
2. Resolve candidate IDs:
   - If `belief_ids is None`, select all node IDs from sorted `nodes.items()` whose `truth_value == "IN"`.
   - Otherwise, filter the supplied IDs to those present in `nodes` and marked `IN`.
3. If no candidate IDs remain, return `[]`.
4. Build `beliefs = {nid: nodes[nid].get("text", "")}` for clustering.
5. Call:
   ```python
   list_clusters(beliefs, model_name=embedding_model or DEFAULT_MODEL)
   ```
6. Set `valid_ids = set(belief_ids)` for later LLM-response filtering.
7. Iterate over `result["clusters"]` with a 1-based cluster index:
   - Extract `cluster_ids = [b["id"] for b in cluster["beliefs"]]`.
   - Skip the cluster if it has fewer than two IDs.
   - Print progress to `stderr`.
   - Chunk `cluster_ids` by `CONTRADICTION_BATCH_SIZE`, which is `50`.
8. For each cluster batch:
   - Format belief text with `format_beliefs_for_contradiction_check(batch, nodes)`, which truncates long belief text to 200 characters.
   - Build the prompt by formatting `CONTRADICTION_PROMPT` with the belief text.
   - Call `invoke_model(prompt, model=model, timeout=timeout)`.
   - Parse the response with `parse_contradiction_response(response, valid_ids=valid_ids)`.
   - Extend `all_results`.
   - Flush results to `output_path` via `_flush_results`.
9. Return `all_results`.

Subtle detail: a cluster of size 51 is split into batches of 50 and 1. The code still invokes the LLM for the final singleton batch, because it only checks the whole cluster size for `< 2`, not each batch size.

### Side Effects

- Prints progress messages to `sys.stderr` per cluster.
- Prints warning messages to `sys.stderr` when a batch-level exception is caught.
- Calls `list_clusters`, which may load/use an embedding model and compute embeddings.
- Calls the LLM through `invoke_model`.
- If `output_path` is set, writes contradiction results incrementally through `_flush_results` and `write_contradiction_plan`.
- Does not mutate `nodes` or the supplied `belief_ids` list.
- No retry logic is implemented.

### Error Handling

Batch-level errors from these calls are caught and converted to warnings:

- `invoke_model(...)`
- `parse_contradiction_response(...)`
- `_flush_results(...)`

The function then continues with the next batch. Therefore a single failed LLM call does not abort the whole run, but it may produce partial results.

Errors are not caught around:

- importing `.cluster`,
- candidate selection,
- `list_clusters(...)`,
- extracting `cluster["beliefs"]` and `b["id"]`,
- `format_beliefs_for_contradiction_check(...)`,
- prompt construction.

Those exceptions propagate to the caller. Also, the `except Exception` does not catch `KeyboardInterrupt` or `SystemExit`.

If `_flush_results` raises, `all_results` has already been extended, but `header_written` is not updated. This can affect subsequent append/overwrite behavior depending on `write_contradiction_plan`.

### Usage Patterns

Typical usage:

```python
nodes = export_network()
results = detect_contradictions_semantic(
    nodes,
    model="claude",
    embedding_model=None,
    output_path="contradiction-plan.md",
)
```

Caller obligations and expectations:

- Pass node data from the expected export format.
- Treat returned results as potentially partial.
- Expect progress and warnings on `stderr`.
- Understand that semantic clustering can miss contradictions between beliefs placed in different clusters.
- Use this variant when semantic grouping is desired; use `detect_contradictions` when fixed-size random batching across all beliefs is desired.
- If `output_path` points to an existing file, be aware that the first successful flush uses `append=False` via `_flush_results`, which likely creates/overwrites depending on `write_contradiction_plan`.

### Dependencies

Internal dependencies:

- `.cluster.list_clusters`
- `.cluster.DEFAULT_MODEL`
- `.llm.invoke_model`
- `CONTRADICTION_BATCH_SIZE`
- `CONTRADICTION_PROMPT`
- `format_beliefs_for_contradiction_check`
- `parse_contradiction_response`
- `_flush_results`
- indirectly `.api.write_contradiction_plan`

Standard library:

- `sys`

External/runtime dependencies:

- Sentence-transformers embedding model used by `list_clusters`.
- The configured LLM backend used by `invoke_model`.

### Assumptions Not Enforced by Types

- `nodes` values are dict-like and support `.get`.
- Node IDs are sortable when `belief_ids is None`.
- `truth_value` is exactly the string `"IN"`.
- Belief text is a string so `len(text)` and slicing work.
- `list_clusters` returns a dict with a sized `"clusters"` list.
- Each cluster has `"beliefs"` entries that are dicts containing `"id"`.
- Cluster IDs correspond to IDs from the original `beliefs` mapping; the function does not re-check cluster IDs against `valid_ids` or `truth_value`.
- `CONTRADICTION_BATCH_SIZE` is a positive integer.
- `invoke_model` accepts the prompt/model/timeout arguments and returns text.
- `parse_contradiction_response` returns the expected result shape.
- The LLM obeys the prompt and uses IDs from the current batch. The parser only filters claims to the global `valid_ids`, so a claim ID from outside the current batch would still be accepted if it is one of the checked IDs.
- `parse_contradiction_response` does not deduplicate claims, so duplicate claim entries can satisfy its `len(claims) >= 2` check.
- `write_contradiction_plan` handles `append=False` and `append=True` as expected.

## Topics to Explore

- [function] `reasons/cluster.py:list_clusters` — Determines cluster structure, embedding model behavior, and whether clusters are disjoint or ordered in a meaningful way.
- [function] `reasons/contradictions.py:detect_contradictions` — The non-semantic variant; compare random batching versus semantic clustering, especially coverage and duplicate behavior.
- [file] `tests/test_contradictions.py` — Likely specifies mocked LLM/cluster behavior and edge cases for filtering and parsing.
- [function] `reasons/api.py:write_contradiction_plan` — Incremental write semantics, append/overwrite behavior, and how partial results appear on disk.
- [general] `semantic-clustering-contradiction-coverage` — The tradeoff between topical grouping and missing contradictions that span different clusters.

## Beliefs

- `semantic-detection-returns-empty-without-in-beliefs` — If no candidate IDs remain after filtering for `truth_value == "IN"`, the function returns `[]` before calling `list_clusters` or the LLM.
- `semantic-detection-filters-claims-by-global-valid-ids` — `parse_contradiction_response` is called with `valid_ids=set(belief_ids)`, so returned claims must be among all checked `IN` beliefs rather than only the current cluster or batch.
- `semantic-detection-swallows-llm-and-flush-exceptions` — Exceptions from `invoke_model`, `parse_contradiction_response`, or `_flush_results` are caught per batch and converted to `stderr` warnings, allowing partial results.
- `semantic-detection-skips-singleton-clusters` — Clusters with fewer than two belief IDs are skipped without an LLM call.
- `semantic-detection-uses-fixed-batch-size` — Within each cluster, beliefs are chunked by the module constant `CONTRADICTION_BATCH_SIZE` (`50`), and the function has no `batch_size` parameter.
