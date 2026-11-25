---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: The names `stored`, `skipped`, `count` being mapped to `stored`, `skipped`, `stored_count` suggests a subtle API mismatch worth verifying in the api layer.

## Explanation

The suspected mismatch is a layer-to-layer schema rename:

- The API function documented at `reasons/api.py:4550` says it returns:
  ```python
  {"stored": [...], "skipped": [...], "count": int}
  ```
  and its return statement at `reasons/api.py:4614-4616` confirms:
  ```python
  "stored": stored,
  "skipped": skipped,
  "count": len(stored),
  ```
  So at the API layer, `count` is an unqualified integer equal to `len(stored)`.

- The MCP wrapper at `reasons/mcp_server.py:306-308` then maps that result into a different public shape:
  ```python
  output["stored"] = store_result["stored"]
  output["stored_count"] = store_result["count"]
  ```
  So MCP clients see `stored_count`, while direct `reasons.api` callers see `count`.

That is not necessarily a bug, but it is a subtle API mismatch worth verifying because:

1. **The same value has two names depending on the caller.**
   Direct API consumers must read `result["count"]`; MCP consumers read `result["stored_count"]`. If any caller assumes the other layer's schema, it will fail.

2. **`count` is ambiguous in the API layer.**
   In the proposal-store function it means "number of stored proposals" because it is set to `len(stored)` at `reasons/api.py:4616`. A reader could easily wonder whether `count` means total processed, total stored, or total stored plus skipped. MCP's `stored_count` is clearer, but that clarity does not exist in the API response contract.

3. **`skipped` remains a list while `stored_count` is an integer.**
   In this function, `skipped` is initialized as a list at `reasons/api.py:4554` and appended to at `4603` and `4608`, then returned at `4615`. So the MCP/API shape is asymmetric: `stored` is a list, `stored_count` is an integer, and `skipped` is a list with no corresponding `skipped_count`.

4. **`skipped` is overloaded elsewhere in `reasons/api.py`.**
   `skipped` is used as an integer counter at lines such as `2135`, `2144`, and `4290`, but as a list in the proposal-store function at `4554`. So the key name alone does not tell consumers whether they are receiving a list or a count.

5. **Test coverage is asymmetric.**
   `tests/test_mcp_server.py:332` asserts `result["stored_count"] == 1`, and `tests/test_mcp_server.py:353` asserts `"stored_count" not in result` in some path. The observed API-level tests around `tests/test_api.py:1499-1619` focus on `result["stored"]`, but the observations do not show an equivalent API-level assertion on `result["count"]`. That leaves the API `count` contract less visibly pinned down.

Also, another consumer, `reasons/cli.py:2866`, reads `store_result["stored"]`, but the observations do not show it reading `count` or `stored_count`. That suggests the rename is mostly an MCP-boundary concern, but it still needs verification at the API boundary.

One observation note: the aggregate `grep_api_stored` result is empty, which is inconsistent with the explicit line-level hits in `reasons/api.py`. The line-level hits are the stronger evidence here.

## Topics to Explore

- [file] `reasons/mcp_server.py` — Inspect the wrapper around lines 270–310 to confirm whether `skipped` is passed through unchanged and whether `stored_count` is always set.
- [file] `reasons/api.py` — Read lines 4544–4616 to verify the full proposal-store function contract and confirm `count` is always `len(stored)`.
- [file] `tests/test_mcp_server.py` — Compare tests at lines 332 and 353 to understand when `stored_count` is present versus omitted.
- [file] `tests/test_api.py` — Check tests around lines 1499–1619 for direct API expectations of `stored`, `skipped`, and `count`.
- [general] `api-response-schema-consistency` — Search all API/MCP responses for `count`, `stored_count`, and `skipped` to see whether this rename is an isolated exception or a broader pattern.

## Beliefs

- `api-count-maps-to-len-stored` — `reasons/api.py:4616` sets the API response key `"count"` to `len(stored)`, so it counts stored proposals.
- `mcp-maps-count-to-stored-count` — `reasons/mcp_server.py:308` maps `store_result["count"]` to `output["stored_count"]`.
- `mcp-passes-through-stored` — `reasons/mcp_server.py:306` passes `store_result["stored"]` through as `output["stored"]` without renaming.
- `proposal-store-skipped-is-list` — The proposal-store API function initializes `skipped` as a list at `reasons/api.py:4554` and appends skipped entries at `4603` and `4608`.
- `api-skipped-has-mixed-types` — `reasons/api.py` uses `skipped` as an integer counter at lines `2135`, `2144`, and `4290`, and as a list in the proposal-store function, so the key's type varies by endpoint.
