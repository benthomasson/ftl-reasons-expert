---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Covers the design pattern of fitting summaries into LLM context windows.

# Design Pattern: Budget-Aware Sectioned Compaction for LLM Context Windows

## The core idea

`reasons/compact.py:compact()` (lines ~15–167) implements a **streaming, priority-ordered budget filler**. It takes an unbounded belief network and produces a Markdown summary that is guaranteed to fit within a caller-supplied token budget, while surfacing the most decision-relevant information first. The same design choice shows up in sibling modules (`reasons/derive.py:_build_beliefs_section`), so it's a project-wide pattern, not a one-off.

## The pattern's building blocks

### 1. Cheap, uniform token estimation (`estimate_tokens`, compact.py:~11`)
```python
def estimate_tokens(text: str) -> int:
    return max(1, len(text) // 4)
```
The `chars/4` BPE heuristic avoids a tokenizer dependency and makes every budget check cheap. Test `TestEstimateTokens.test_uses_char_count_not_word_count` locks in the semantic (chars, not words). The `max(1, ...)` floor prevents empty fragments from being treated as free.

### 2. Priority ordering of sections
The docstring codifies the priority:
1. **Nogoods** — contradiction records, highest signal-per-byte
2. **OUT nodes** — retracted beliefs that need review (annotated with `stale_reason` / `superseded_by` at lines ~106–113)
3. **IN nodes** — active beliefs sorted by `len(node.dependents)` descending (`in_nodes.sort(...)`, line ~41)

Tests `test_budget_limits_nogoods` / `test_budget_limits_out_nodes` / `test_budget_limits_in_nodes` each verify the section-specific overflow marker fires. `test_most_depended_on_first` verifies the ordering within Section 3.

### 3. Footer reservation
```python
footer_tokens = estimate_tokens(f"Token count: ~{budget} / {budget} budget")
```
The final "Token count" line is *pre-charged* against the budget so the file never violates its own promise. This is why `_over_budget(line)` includes `+ footer_tokens` — every candidate line is checked against budget **net of the tail cost**.

### 4. O(1) incremental budgeting
Rather than `"\n".join(lines)` and re-measuring on every append (`O(n²)` behavior), the code keeps a running `_char_count` and computes `_current_tokens()` as `max(1, _char_count // 4)`. `_add_line` does the arithmetic:
```python
_char_count += 1 + len(line)   # +1 for the \n separator
```
This matters because each node candidate is checked twice (once via `_over_budget`, once appended) across potentially hundreds of nodes.

### 5. Per-line admission control with explicit omission
The guard at each iteration:
```python
if _over_budget(line):
    remaining = len(visible_nodes) - added
    _add_line(f"  ... ({remaining} more IN nodes omitted)")
    break
```
Two important properties:
- **Header-level gate** (`if network.nogoods and not _over_budget("## Nogoods")`) — an empty section isn't emitted at all if even the header won't fit.
- **Omission marker** — the reader/LLM always learns *how much* is missing. Silent dropping would make the summary misleading; `... (N more IN nodes omitted)` makes the compression transparent. This is what `test_budget_respected_across_all_sections` depends on.

### 6. Two-level abstraction via summary nodes
Section 3 introduces a secondary consideration: individual nodes may be covered by a **summary node** (one with `metadata["summarizes"] = [ids...]`). The code splits IN nodes into `summary_nodes` and `regular_nodes`, builds `covered_by_summary`, then filters:
```python
visible_nodes = summary_nodes + [n for n in regular_nodes if n.id not in covered_by_summary]
```
The result is that summaries are always shown (with `[summary]` prefix and `(covers N nodes)` suffix) and the nodes they abstract over are suppressed from the visible set — while `hidden_count` records the savings:
```python
if hidden_count:
    _add_line(f"  ({hidden_count} nodes hidden by summaries)")
```
This is the layer that lets a caller invest once in a summary (via `api.summarize`, `reasons/api.py:1141`) to free up downstream context budget.

### 7. Local truncation as a second compression axis
`_text(node)` truncates to 80 chars (`t[:77] + "..."`) when `truncate=True`. Combined with cheap annotations, each node line becomes a fixed-cost "hook" — enough for the LLM to recognize the belief without paying for its full text:
```python
line = f"- {prefix}{node.id}: {_text(node)}{deps}{dep_info}{sum_info}"
```
`test_truncates_long_text` / `test_no_truncate` pin both modes.

## Signal density choices

Each line carries a small set of **cheap prioritization telegraphs** rather than prose:
- `<- antecedent1, antecedent2` — first non-summarizes justification (compact.py:~148)
- `(N dependents)` — impact fan-out
- `(covers N nodes)` — summary breadth
- ` (stale: ...)` / ` (superseded by: ...)` — lifecycle cause on OUT nodes

Tests `test_shows_dependencies`, `test_shows_dependent_count`, `test_stale_reason_in_out` cover each of these.

## Where the pattern lives elsewhere

The **same pattern, different allocator** appears in `reasons/derive.py:_build_beliefs_section` (validated by `tests/test_derive_budget.py`). There, instead of one global budget with a streaming filler, the code does **proportional allocation with a floor**: agent beliefs take `max(5, int(max_beliefs * agent_n / total))`, locals get `max(5, max_beliefs - count)`. The test file documents a historical bug (`count += len(belief_ids)` inside the per-belief loop → `N²` instead of `N`), which is exactly the kind of state-tracking failure the incremental `_char_count` in `compact.py` avoids for the token budget.

Two sibling surfaces consume the compact pattern:
- `reasons/api.py:2869` — `def compact(budget=500, truncate=True, visible_to=None, ...)` wraps it for programmatic callers.
- `reasons/cli.py:1254` — `cmd_compact` invokes `api.compact` and prints the summary for e.g. `CLAUDE.md` injection.

## What the observations don't cover

- The exact call path from `compact()` output into `CLAUDE.md` (the docstring mentions the target, but no writer code is in the excerpt).
- Whether `budget` is meant to be checked against a *specific model's* context window or a shared target — `find_context_window_refs` shows no `max_tokens` tie-in at this layer, suggesting budgets are caller-chosen.
- The `visible_to` discriminator on `api.compact` (line 2869) implies a multi-actor visibility layer that isn't visible in the core function.

## Topics to Explore

- [function] `reasons/compact.py:compact` — Compare the budget-check math (`_over_budget`) with the older proportional allocator to see why streaming admission beats static percentages for heterogeneous sections.
- [file] `tests/test_derive_budget.py` — The quadratic-count regression documents what goes wrong when budget tracking is nested inside the wrong loop.
- [function] `reasons/api.py:summarize` — The upstream primitive that produces `metadata["summarizes"]` relationships the compactor relies on to hide covered nodes.
- [function] `reasons/api.py:compact` — The public wrapper with `visible_to`, `include_out`, and `include_antecedents` flags — reveals which knobs the CLI and ask-pipeline actually need.
- [general] `summary-node-lifecycle` — How summary nodes are created, validated (`label != "summarizes"` guard in compact.py), and invalidated when their covered set changes.

## Beliefs

- `compact-produces-budget-bounded-summary` — `compact(network, budget)` never emits more than `budget + small structural overhead` tokens because `_over_budget` reserves `footer_tokens` before admitting any line.
- `estimated-tokens-is-a-property-of-compact` — `estimate_tokens` uses `len(text) // 4` with a floor of 1; the whole budget system inherits this units convention and cannot be swapped for a real tokenizer without changing every `_over_budget` call site.
- `omission-markers-are-a-hard-contract` — Every truncated section emits an `... (N more X omitted)` line; the tests `test_budget_limits_nogoods`, `test_budget_limits_out_nodes`, `test_budget_limits_in_nodes` all assert on this marker, so callers can rely on absence = completeness.
- `summary-nodes-hide-covered-nodes` — When a node has `metadata["summarizes"]`, its covered IDs are removed from `visible_nodes` and counted in `hidden_count`; this is the only mechanism by which compaction can present *more* beliefs than the direct line budget would allow.
- `in-node-ordering-is-stable-by-dependent-count` — `in_nodes.sort(key=lambda n: len(n.dependents), reverse=True)` is applied *twice* (once before Section 2, once again inside Section 3 on `visible_nodes`), so the cheapest admission policy is always "most-depended-on first."
- `budget-check-requires-footer-tokens` — Removing `+ footer_tokens` from `_over_budget` would allow the final `Token count: ~N / N budget` line to exceed the declared budget for small budgets, breaking `test_budget_respected_across_all_sections`.
