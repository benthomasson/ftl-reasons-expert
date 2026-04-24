# Code Review Report

**Branch:** benthomasson/ftl-reasons#5
**Models:** claude, gemini
**Gate:** [BLOCK] BLOCK

## claude [CONCERN]

### reasons_lib/derive.py:_tokenize_id
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The function replaces `:` with `-` before splitting. This means namespace prefixes like `agent-a:gl108-disabled` tokenize to `{"agent", "a", "gl108", "disabled"}` — the namespace tokens mix into the belief tokens. Two beliefs from different namespaces with similar base IDs will match, which may be desirable but also means two completely different beliefs `ns1:foo-bar` and `ns2:foo-bar` would have 100% similarity. Since namespaces were added in this same PR, this interaction deserves a test. Additionally, common short tokens like single-character namespace fragments ("a") could inflate Jaccard similarity for unrelated beliefs.

### reasons_lib/derive.py:_jaccard
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Standard Jaccard implementation. Handles empty sets correctly.

### reasons_lib/derive.py:find_similar_out
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Correctly filters to OUT nodes, computes similarity, sorts by similarity descending. Uses dict-style access (`node.get("truth_value")`) which is consistent with how `nodes` is passed from callers (as plain dicts from the api layer).

### reasons_lib/derive.py:validate_proposals (modified)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

This is the core fix. After the existing "already exists" check, it now calls `find_similar_out` to detect variant IDs of retracted beliefs. The skip message includes the matched ID and overlap percentage, which is good for auditability. The ordering is correct: exact-match check first, then fuzzy check, which avoids unnecessary similarity computation.

### reasons_lib/api.py:_resolve_namespace
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** WIRED

The `:` check as a proxy for "already namespaced" is fragile. If a belief ID legitimately contains a colon for non-namespace reasons, it won't get prefixed. No tests exist for this function. The function is called in `add_node` on both the node_id and on antecedent/outlist references — if a user passes `--sl "foo,bar"` with `--namespace agent1`, the antecedents `foo` and `bar` get prefixed to `agent1:foo` and `agent1:bar`, but the original SL antecedent string was parsed without namespace awareness, so the caller must know to pass un-prefixed IDs. This implicit contract is undocumented.

### reasons_lib/api.py:ensure_namespace
**Verdict:** CONCERN
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** PARTIAL

Function is correct but has no dedicated test and no CLI command wires to it. It's only used indirectly through `add_node`'s namespace path. The function duplicates the logic in `add_node` for creating the namespace premise — if one is updated, the other could drift. Consider extracting the common logic.

### reasons_lib/api.py:list_namespaces
**Verdict:** CONCERN
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** WIRED

Logic is straightforward. Wired to CLI via `cmd_namespaces`. However, no tests exist for this function. The namespace detection heuristic (`:active` suffix + `agent_premise` role) is correct for programmatically-created namespaces but brittle — a manually created node ending in `:active` without the metadata would be missed silently, which is fine. Counting with `startswith` is O(n) over all nodes for each namespace — acceptable at current scale.

### reasons_lib/api.py:add_node (modified, namespace support)
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** WIRED

The namespace logic has a subtle issue. When `namespace` is set and no explicit justification is provided, a new SL justification is created with `antecedents=[active_id]` and `outlist=outlist`. Then lines 184-186 resolve namespaces in all antecedents/outlist — but `active_id` is already fully qualified (e.g., `agent:active`), and `_resolve_namespace` skips it because it contains `:`. So that's fine. However, consider: if `sl="dep-a,dep-b"` and `namespace="agent"`, the antecedents are parsed as `["dep-a", "dep-b"]`, then `active_id` is prepended, then `_resolve_namespace` runs on all three — `active_id` is skipped (has `:`), but `dep-a` becomes `agent:dep-a`. This assumes `dep-a` exists as `agent:dep-a` in the network, which may not be true if the dependency is cross-namespace. The `_resolve_namespace` function handles this by skipping IDs that already contain `:`, but there's no test coverage for this cross-namespace scenario.

### reasons_lib/api.py:deduplicate
**Verdict:** CONCERN
**Correctness:** QUESTIONABLE
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

The union-find approach is correct for clustering. However, there's a **transitive clustering concern**: if A is similar to B (0.6) and B is similar to C (0.6), but A is not similar to C (0.2), all three end up in the same cluster. With `--auto`, C could be retracted despite being quite different from A. This is a known property of union-find clustering but may surprise users. The O(n²) comparison is fine for typical network sizes but could be slow at 5,500+ nodes as mentioned in the issue. The `keep` selection uses `max(members, key=lambda nid: (len(net.nodes[nid].dependents), nid))` — the tiebreaker is lexicographic by ID, which is deterministic but arbitrary. Tests cover the basic cases well.

### reasons_lib/api.py:list_nodes (modified)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** WIRED

Simple filter addition. Uses `startswith` to match namespace prefix, which is consistent with the namespace convention. Wired to CLI via `--namespace/-n` flag.

### reasons_lib/cli.py:cmd_deduplicate
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** PARTIAL
**Integration:** WIRED

Clean CLI handler. Output formatting is clear with cluster info, kept/retracted status. No CLI-level tests (would need to invoke via subprocess or `main()`), but the underlying API function is tested.

### reasons_lib/cli.py:cmd_namespaces
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** UNTESTED
**Integration:** WIRED

Simple display function. Properly wired to CLI dispatch table.

### reasons_lib/cli.py (parser additions)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** PARTIAL
**Integration:** WIRED

All new subcommands (`deduplicate`, `namespaces`) and flags (`--namespace` on `add` and `list`, `--threshold`, `--auto`) are correctly wired to the dispatch table. The `getattr(args, "namespace", None)` pattern in `cmd_add` and `cmd_list` is safe — the attribute exists because argparse sets it, but `getattr` with default is defensive against commands that don't have the flag.

### tests/test_derive.py (duplicate detection tests)
**Verdict:** PASS
**Correctness:** VALID
**Spec Compliance:** N/A
**Test Coverage:** COVERED
**Integration:** WIRED

Comprehensive test coverage for the core fix. Tests cover: tokenization (with and without namespaces), Jaccard edge cases (identical, disjoint, partial, empty), `find_similar_out` (positive match, IN-only ignored, no match), `validate_proposals` (similar-to-retracted rejected, unrelated allowed), and `deduplicate` (cluster detection, no-cluster case, auto-retract with verification).

### Self-Review
**Limitations:** Could not run the test suite to verify all tests pass. Did not see the full `cmd_accept` or `cmd_derive` code paths to verify the guard is invoked during actual accept/derive workflows (only verified `validate_proposals` is called from `derive.py`). Did not check whether `deduplicate`'s O(n²) is actually problematic at 5,500 nodes. Could not verify whether `apply_proposals` in the accept flow calls `validate_proposals` — if accept bypasses validation, the guard wouldn't help for the accept path described in the issue.

### Feature Requests
- Include callers of modified functions (e.g., who calls `validate_proposals`) so integration can be verified end-to-end
- Flag when a diff contains untested code alongside tested code — makes the "shipped without tests" pattern visible
- Detect scope creep: when a diff contains changes unrelated to the stated issue, flag them separately

## gemini [BLOCK]

### ERROR
**Verdict:** BLOCK

Model invocation failed: Model gemini failed: Error when talking to Gemini API Full report available at: /var/folders/nj/ypj45mm145j6dttg0sz0t02h0000gn/T/gemini-client-error-Turn.run-sendMessageStream-2026-04-17T18-52-41-303Z.json TypeError: fetch failed
    at node:internal/deps/undici/undici:16416:13
    at async Models.generateContentStream (file:///Users/ben/.npm-global/lib/node_modules/@google/gemini-cli/node_modules/@google/genai/dist/node/index.mjs:13358:24)
    at async file:///Users/ben/.npm-global/lib/node_modules/@google/gemini-cli/node_modules/@google/gemini-cli-core/dist/src/core/loggingContentGenerator.js:256:26
    at async file:///Users/ben/.npm-global/lib/node_modules/@google/gemini-cli/node_modules/@google/gemini-cli-core/dist/src/telemetry/trace.js:81:20
    at async retryWithBackoff (file:///Users/ben/.npm-global/lib/node_modules/@google/gemini-cli/node_modules/@google/gemini-cli-core/dist/src/utils/retry.js:130:28)
    at async GeminiChat.makeApiCallAndProcessStream (file:///Users/ben/.npm-global/lib/node_modules/@google/gemini-cli/node_modules/@google/gemini-cli-core/dist/src/core/geminiChat.js:445:32)
    at async GeminiChat.streamWithRetries (file:///Users/ben/.npm-global/lib/node_modules/@google/gemini-cli/node_modules/@google/gemini-cli-core/dist/src/core/geminiChat.js:265:40)
    at async Turn.run (file:///Users/ben/.npm-global/lib/node_modules/@google/gemini-cli/node_modules/@google/gemini-cli-core/dist/src/core/turn.js:70:30)
    at async GeminiClient.processTurn (file:///Users/ben/.npm-global/lib/node_modules/@google/gemini-cli/node_modules/@google/gemini-cli-core/dist/src/core/client.js:477:26)
    at async GeminiClient.sendMessageStream (file:///Users/ben/.npm-global/lib/node_modules/@google/gemini-cli/node_modules/@google/gemini-cli-core/dist/src/core/client.js:577:20) {
  [cause]: HeadersTimeoutError: Headers Timeout Error
      at FastTimer.onParserTimeout [as _onTimeout] (/Users/ben/.npm-global/lib/node_modules/@google/gemini-cli/node_modules/undici/lib/dispatcher/client-h1.js:749:28)
      at Timeout.onTick [as _onTimeout] (/Users/ben/.npm-global/lib/node_modules/@google/gemini-cli/node_modules/undici/lib/util/timers.js:162:13)
      at listOnTimeout (node:internal/timers:605:17)
      at process.processTimers (node:internal/timers:541:7) {
    code: 'UND_ERR_HEADERS_TIMEOUT'
  }
}
An unexpected critical error occurred:[object Object]

