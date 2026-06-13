# compact

[Back to index](index.md)

The `compact` module provides a deterministic, budget-aware serialization of a belief network into a markdown string. It acts as an information distillation layer — reducing a full `Network` into a concise textual summary suitable for context-limited consumers like LLM prompts, while preserving structurally important beliefs and critical diagnostic information such as contradictions.

## Pure Transformation

The `compact()` function is a pure transformation from `Network` to `str` (`compact-is-pure-function`, `compact-pure-function`). It performs no I/O, no database access, and no mutations against the network instance. This purity makes its output fully deterministic for a given input — the same network and budget always produce the same string.

## Output Structure

The output is organized into three markdown sections: `## IN (active)`, `## OUT (retracted)`, and `## Nogoods`, each present only when the network contains nodes of that type (`compact-three-sections`). These sections are emitted in a fixed priority order: nogoods first, then OUT nodes, then IN nodes (`compact-priority-order`, `compact-priority-order-is-nogoods-out-in`). This ordering ensures that contradictions and retracted beliefs — typically the most diagnostically valuable information — are never displaced by active beliefs. If the budget runs out mid-emission, lower-priority sections are omitted entirely rather than partially filled.

## Budget Management

### Approximate, Not Exact

Early analysis held that the budget was a hard ceiling (`compact-never-exceeds-budget`), but this understanding was refined: the token budget is approximate, and structural overhead such as section headers and truncation messages can cause output to exceed the specified budget by up to roughly 25% (`compact-budget-is-soft-cap`, `compact-budget-soft-ceiling`). The belief that the budget reliably constrains total output size has been retracted (`compact-budget-controls-output-size`), along with the claim that compact is simultaneously deterministic, bounded, and self-describing without trade-offs (`compact-is-deterministic-pure-and-bounded`, `compact-is-efficient-deterministic-and-bounded`). The function is deterministic and pure, but "bounded" requires the caveat of structural overhead.

An earlier belief that token estimation was word-count-based has also been retracted (`compact-token-estimate-is-word-count`). The actual implementation uses `len(text) // 4` with a floor of 1 (`compact-estimate-tokens-chars-div-4`) — a lightweight approximation that avoids external tokenizer dependencies while remaining computationally cheap.

### Efficient Tracking

Budget tracking uses a running `_char_count` integer, making per-line budget checks O(1) rather than scanning the accumulated output (`compact-char-tracking-o1`). Combined with the chars/4 estimation, this gives the compact module an efficient and approximate budget enforcement strategy (`compact-budget-tracking-is-efficient-and-approximate`). The footer line's token cost is pre-computed and reserved before any section begins filling, guaranteeing the auditability metadata always fits (`compact-footer-pre-reserved`).

When a budget is specified, the output includes a `Token count: N / B budget` line for auditability (`compact-self-reports-tokens`).

## Priority and Ordering

Within the IN nodes section, nodes are sorted by descending dependent count so that structurally important beliefs — those depended on by many others — are emitted first and survive budget truncation (`compact-in-nodes-ordered-by-dependents`). This means that foundational premises and heavily-referenced derived beliefs take priority over leaf nodes when space is constrained.

## Summary Node Elision

When a node's `summarizes` metadata lists other node IDs, those covered nodes are excluded from compact output to avoid redundancy; the count of hidden nodes is reported (`compact-summary-nodes-hide-covered`). Critically, this elision only applies when the summary node itself is IN — if the summary is retracted, all covered nodes reappear in the output (`compact-summary-hiding-requires-in`). This prevents information loss when summaries become invalid.

## Robustness

The function handles empty networks, zero-budget inputs, and missing metadata without raising exceptions — it is designed to always produce a valid string (`compact-is-infallible`). Together with the priority ordering and pre-reserved footer, this makes the compact module a structurally complete and infallible serializer (`compact-structure-is-priority-ordered-and-infallible`).

OUT nodes that carry `stale_reason` metadata include the reason string in the output (`compact-surfaces-stale-reason-metadata`), ensuring that staleness information survives the binary IN/OUT truth model and reaches downstream consumers.

## PostgreSQL Variant

The `PgApi.compact()` method extends the core behavior with database-backed features: it produces a markdown summary constrained by a token budget, filtered by `visible_to` access tags, sorted by dependent count, with summary-node elision and nogood inclusion (`pg-compact-generates-budget-constrained-markdown`). This variant bridges the pure in-memory compaction with the access-controlled persistence layer.
