# compact

[Back to index](index.md)

The `compact` function is a pure, deterministic transformation that serializes a belief network into a budget-constrained markdown summary (`compact-is-pure-function`, `compact-pure-function`). It performs no I/O, no database access, and no mutations — taking a `Network` as input and returning a plain string. The function is designed to always succeed, handling empty networks, zero budgets, and missing metadata without raising exceptions (`compact-is-infallible`).

## Output Structure

The compact output is organized into three markdown sections — `## IN (active)`, `## OUT (retracted)`, and `## Nogoods` — each present only when the network contains nodes of that type (`compact-three-sections`). These sections are emitted in strict priority order: nogoods first, then OUT nodes, then IN nodes (`compact-priority-order-is-nogoods-out-in`). A later section never displaces content from an earlier one; if the budget is exhausted early, lower-priority sections are omitted entirely (`compact-priority-order`).

The footer line's token cost is pre-computed and reserved before any section begins filling, guaranteeing it always fits within the budget (`compact-footer-pre-reserved`). Together, the priority ordering, edge-case handling, and footer reservation make the output structurally complete and predictably bounded (`compact-structure-is-priority-ordered-and-infallible`).

## Budget Enforcement

The `compact` function enforces a token budget by pre-checking every line addition against remaining space (`compact-never-exceeds-budget`). Budget tracking is efficient: a running `_char_count` integer makes per-line checks O(1) rather than O(n) (`compact-char-tracking-o1`), and token estimation uses `len(text) // 4` with a floor of 1 — a lightweight approximation that avoids external tokenizer dependencies (`compact-estimate-tokens-chars-div-4`). This combination yields efficient, approximate budget tracking sufficient for enforcement (`compact-budget-tracking-is-efficient-and-approximate`).

In practice, the budget is a soft ceiling. Structural overhead — section headers, truncation messages — can cause the output to exceed the specified budget by up to approximately 25% (`compact-budget-is-soft-cap`, `compact-budget-soft-ceiling`). When a budget is provided, the output includes a `Token count: N / B budget` line for auditability (`compact-self-reports-tokens`).

## Node Ordering and Filtering

Within the IN section, nodes are sorted by descending dependent count so that structurally important nodes — those depended on by many others — appear first and survive budget truncation (`compact-in-nodes-ordered-by-dependents`).

Summary nodes provide a deduplication mechanism: IN nodes whose IDs appear in another node's `summarizes` metadata list are excluded from the output to avoid redundancy, with the hidden count reported (`compact-summary-nodes-hide-covered`). This hiding is conditional — a summary node only suppresses its covered nodes when the summary itself is IN. If the summary goes OUT, the covered nodes reappear in the compact output (`compact-summary-hiding-requires-in`).

For OUT nodes, those carrying `stale_reason` metadata include the reason string in the output, preserving staleness information that would otherwise be lost in the binary IN/OUT truth model (`compact-surfaces-stale-reason-metadata`).

## Database-Backed Variant

The PostgreSQL API exposes `PgApi.compact()`, which produces the same budget-constrained markdown summary but adds access control via `visible_to` tags, filtering the network before serialization (`pg-compact-generates-budget-constrained-markdown`). It retains the same sorting, summary-node elision, and nogood inclusion behavior as the core function.
