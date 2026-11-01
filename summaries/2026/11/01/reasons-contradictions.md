---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/contradictions.py

# `reasons/contradictions.py`

## 1. Purpose

This module is the **contradiction detector** in the `ftl-reasons` belief-network system. Its single responsibility: take a set of currently-held (truth_value = `"IN"`) beliefs, send them to an LLM, and extract **nogoods** — sets of beliefs that are logically incompatible and cannot all be true simultaneously.

It's the "audit" step in the belief lifecycle. Other modules (`repair.py`, `review.py`, `derive.py`) act on beliefs; this one *finds problems* between them. The results feed into the broader repair/defeat pipeline via `write_contradiction_plan` in `api.py`.

## 2. Key Components

### Constants

| Name | Value | Role |
|------|-------|------|
| `CONTRADICTION_BATCH_SIZE` | `50` | Max beliefs per LLM call. Prevents prompt overflow and keeps the LLM's context focused. |
| `CONTRADICTION_PROMPT` | (string) | The full system prompt. Defines 4 contradiction types (direct negation, incompatible properties, scope conflict, quantitative mismatch) and enforces a strict output format (`### NOGOOD <id>` blocks). |

### Functions

**`format_beliefs_for_contradiction_check(belief_ids, nodes) -> str`**
- Truncates each belief's text to 200 chars (197 + `"..."`).
- Produces a flat `- \`nid\`: text` list for prompt interpolation.
- Silently skips IDs not present in `nodes`.

**`parse_contradiction_response(response, valid_ids=None) -> list[dict]`**
- Regex-based parser for the LLM's structured output.
- Returns a list of `{"id", "claims", "analysis", "severity"}` dicts.
- **Filtering**: if `valid_ids` is provided, any claim not in that set is dropped. A nogood with fewer than 2 surviving claims is discarded entirely.
- This is the critical validation gate — it prevents hallucinated belief IDs from leaking into the belief graph.

**`_flush_results(results, output_path, header_written) -> bool`**
- Lazily imports `write_contradiction_plan` from `.api` (avoids circular import at module load).
- Writes incrementally; the `header_written` boolean ensures the file header is written exactly once across multiple batches.
- Returns the updated `header_written` flag.

**`detect_contradictions(nodes, belief_ids=None, model="claude", timeout=300, batch_size=50, output_path=None) -> list[dict]`**
- **Primary entry point.** Random-shuffles all IN beliefs, splits into batches of `batch_size`, and fires LLM calls sequentially.
- Random shuffling is intentional: it prevents the LLM from missing contradictions that only appear when specific beliefs land in the same batch (a single batch of 50 out of 500 will miss cross-batch pairs).
- Returns a flat list of all nogood dicts across all batches.

**`detect_contradictions_semantic(nodes, belief_ids=None, model="claude", timeout=300, embedding_model=None, output_path=None) -> list[dict]`**
- **Alternative entry point.** Instead of random batching, it first calls `list_clusters` (from `.cluster`) to group beliefs by semantic similarity using sentence-transformers embeddings, *then* runs the same LLM check within each cluster.
- Rationale: two beliefs about "quantum computing error correction" are far more likely to contradict each other than a belief about "quantum computing" and one about "baking bread." Clustering increases recall.
- Still applies `CONTRADICTION_BATCH_SIZE` within each cluster (a cluster might be larger than 50).

## 3. Patterns

- **Lazy imports for heavy/circular deps**: `write_contradiction_plan` and `list_clusters` are imported inside function bodies, not at module top-level. This breaks the potential `contradictions → api → contradictions` cycle and defers loading sentence-transformers until actually needed.
- **Structured-LLM-output + regex parse**: The prompt demands a very specific `### NOGOOD` format, and the parser is a single compiled regex (`_NOGOOD_PATTERN`) with `re.DOTALL`. This is a pragmatic middle ground — not as strict as JSON mode, but more flexible than free-text.
- **Incremental I/O**: `_flush_results` writes after each batch so a long-running multi-batch job doesn't lose everything if it crashes mid-way.
- **Two strategies, same core**: `detect_contradictions` and `detect_contradictions_semantic` share `format_beliefs_for_contradiction_check`, `CONTRADICTION_PROMPT`, `parse_contradiction_response`, and `_flush_results`. They differ only in *how they group beliefs into batches* (random vs. semantic clusters).
- **Defensive filtering**: `valid_ids` set is threaded through parsing to reject hallucinated IDs. Text truncation prevents prompt blowup. `belief_ids` is re-filtered against `nodes` to drop stale IDs.

## 4. Dependencies

### This file imports

| Import | Why |
|--------|-----|
| `random` | Shuffle belief order before batching |
| `re` | Compile `_NOGOOD_PATTERN` for response parsing |
| `sys` | Progress/warning messages to `stderr` |
| `.llm.invoke_model` | The LLM call primitive (model-agnostic wrapper) |
| `.api.write_contradiction_plan` *(lazy)* | Persist results to a plan file |
| `.cluster.list_clusters`, `.cluster.DEFAULT_MODEL` *(lazy)* | Semantic clustering for the variant entry point |

### This file is imported by

| Importer | Relationship |
|----------|-------------|
| `reasons/api.py` | The public API surface; `api.py` likely exposes `detect_contradictions` as a higher-level endpoint |
| `reasons_lib/contradictions.py` | Mirror/duplicate copy (the repo has a `reasons/` and a `reasons_lib/` tree — likely a packaging artifact) |
| `tests/test_contradictions.py` | Unit/integration tests for parsing, batching, and end-to-end flow |

## 5. Flow

```
detect_contradictions(nodes, belief_ids)
│
├─ 1. Resolve belief_ids
│     default: all nodes where truth_value == "IN"
│     custom:  intersect provided IDs with nodes, keep only "IN"
│
├─ 2. Early exit if empty
│
├─ 3. random.shuffle(belief_ids)          ← random strategy
│
├─ 4. Loop: for each batch of 50
│     ├─ format_beliefs_for_contradiction_check(batch, nodes)
│     ├─ CONTRADICTION_PROMPT.format(beliefs=...)
│     ├─ invoke_model(prompt, model, timeout)
│     ├─ parse_contradiction_response(response, valid_ids)
│     │     └─ regex → extract NOGOOD blocks → filter claims → drop <2 claims
│     ├─ all_results.extend(results)
│     └─ _flush_results(results, output_path, header_written)
│
└─ return all_results
```

`detect_contradictions_semantic` replaces step 3 with:

```
│  3'. list_clusters(beliefs, model_name=embedding_model)
│     └─ returns {"clusters": [{"beliefs": [{"id", "text"}]}, ...]}
│
│  4'. Loop: for each cluster (skip if <2 beliefs)
│       └─ for each sub-batch of 50 within the cluster
│            └─ (same prompt → invoke → parse → flush as above)
```

## 6. Invariants

| Invariant | Enforced by |
|-----------|-------------|
| Only `truth_value == "IN"` beliefs are ever checked | Both `detect_*` functions filter at entry |
| Every returned nogood has **≥ 2** claims | `parse_contradiction_response` drops entries where `len(claims) < 2` |
| Every claim ID in a returned nogood exists in the input `nodes` | `valid_ids` filtering in the parser |
| Belief text in the prompt is **≤ 200 chars** | Truncation in `format_beliefs_for_contradiction_check` |
| The output file header is written **at most once** | `header_written` boolean threaded through `_flush_results` |
| `CONTRADICTION_BATCH_SIZE` is the upper bound on beliefs per LLM call | Both loop structures chunk at this constant |

## 7. Error Handling

The error strategy is **degrade-and-continue**, not fail-fast:

- **Per-batch LLM failures** (timeout, API error, network blip): caught by the bare `except Exception as e` in both `detect_contradictions` and `detect_contradictions_semantic`. A `WARN` line is printed to `stderr`, the batch's results are simply empty, and the loop moves to the next batch. The function still returns a partial result list.
- **Malformed LLM output** (missing NOGOOD headers, wrong field names): `parse_contradiction_response` simply doesn't match anything — no exception, no partial parse. The result list for that batch is empty.
- **Unknown belief IDs in LLM output**: filtered out by `valid_ids` membership check.
- **Missing node data**: `format_beliefs_for_contradiction_check` does `nodes.get(nid)` and `continue`s if `None`.
- **Empty belief set**: both entry points return `[]` immediately before any LLM call.

Notably, there is **no retry logic**. A failed batch is lost. If you need reliability under flaky LLM APIs, that would be an addition in `invoke_model` or here.

One thing to watch: the `except Exception` is broad. A `KeyboardInterrupt` (which inherits from `BaseException`, not `Exception`) will still propagate, which is correct. But a programming bug like a `TypeError` in the parser would be silently swallowed and reported as a generic batch failure.

---

## Topics to Explore

- [file] `reasons/cluster.py` — Implements `list_clusters` and `DEFAULT_MODEL`; understanding how beliefs are semantically grouped is essential to evaluating `detect_contradictions_semantic`'s recall advantages.
- [function] `reasons/api.py:write_contradiction_plan` — The sink for contradiction results. Knowing its file format and append semantics is critical for debugging incremental output and for the downstream repair pipeline that consumes these plans.
- [file] `tests/test_contradictions.py` — Shows the expected parsing behavior, edge cases (empty responses, hallucinated IDs, single-claim nogoods), and the mocking strategy for `invoke_model`.
- [file] `reasons/llm.py` — Wraps `invoke_model`; understanding its retry behavior (or lack thereof), model routing, and timeout semantics directly affects how `contradictions.py` behaves under production load.
- [general] `reasons-vs-reasons_lib-duplication` — The repo ships both `reasons/` and `reasons_lib/` with near-identical files. Understanding whether this is a build/packaging step, a legacy artifact, or intentional dual-entry-point design will save confusion when editing one and forgetting the other.

---

## Beliefs

- `contradictions-checks-only-IN-beliefs` — Both `detect_contradictions` and `detect_contradictions_semantic` filter to `truth_value == "IN"` at entry; beliefs in any other state (OUT, DEFENDED, etc.) are never sent to the LLM.
- `parse-rejects-nogoods-with-fewer-than-2-claims` — `parse_contradiction_response` drops any NOGOOD block whose `claims` list has length < 2 after `valid_ids` filtering, guaranteeing every returned nogood is a genuine multi-belief conflict.
- `detect-contradictions-shuffles-before-batching` — `random.shuffle(belief_ids)` is called before the batch loop in `detect_contradictions`, meaning batch composition is non-deterministic across runs; the same set of beliefs can produce different contradiction sets depending on which IDs land together.
- `semantic-variant-uses-embedding-model-not-llm-model` — `detect_contradictions_semantic` takes a separate `embedding_model` parameter (defaulting to `cluster.DEFAULT_MODEL`) for clustering, distinct from the `model` parameter used for the LLM contradiction check; mixing these up would either break clustering or send embeddings to the wrong endpoint.
- `incremental-write-header-once` — The `header_written` boolean ensures `write_contradiction_plan`'s header is emitted at most once per `output_path` across all batches; a bug that resets this flag would produce duplicate headers in the output file.
