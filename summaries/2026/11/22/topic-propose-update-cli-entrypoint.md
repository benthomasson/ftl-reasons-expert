---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-22
---

# Topic: Find the CLI command that invokes `propose_updates`; a `basis`-based auto-accept gate would have to live here (e.g. skip markdown for `source-divergence`, keep it for `prior-knowledge`).

## Where the CLI invokes `propose_updates`

The CLI → library call chain has one intermediate hop, so `propose_updates` is **not** called directly from the CLI — it's wrapped by `api.propose_update`. Full trace:

### 1. Command registration — `reasons/cli.py`
- **Line 3801**: `p = sub.add_parser("propose-update", ...)` — the subcommand name is `propose-update`.
- **Line 4007**: `"propose-update": cmd_propose_update,` — dispatcher maps that subcommand to the handler.

### 2. Handler — `cmd_propose_update` — `reasons/cli.py:2775`
```
def cmd_propose_update(args):        # line 2775
    _require_sqlite(args, "propose-update")   # line 2776
    from .propose_update import format_proposals_file   # line 2782
    report_path = report_dir / f"propose-update-{ts}.json"  # line 2793
    result = api.propose_update(      # line 2804
```
So the CLI handler shells out to `api.propose_update(...)` and then writes a report at line 2793.

### 3. API wrapper — `api.propose_update` — `reasons/api.py:4458`
- **Line 4475**: `from .propose_update import propose_updates as _propose` — the actual library call is aliased here.
- **Line 4544**: `_store_proposals` converts “LLM propose_update output into stored proposals”.
- **Line 4561**: `basis = p.get("basis", "prior-knowledge")` — this is where each proposal’s `basis` is read back out and persisted with the proposal row.

### 4. Real implementation — `propose_updates` — `reasons/propose_update.py:291`
- `BASES = {"source-divergence", "detected-contradiction", "prior-knowledge"}` is defined near the top of the module.
- `parse_update_proposals` normalizes each proposal’s `basis` (unknown → `prior-knowledge`), so by the time `propose_updates` returns, every proposal carries a valid `basis` in the set above.

### Where the `basis`-gated auto-accept belongs

Because `basis` is already populated on each proposal object returned by `propose_updates` / persisted by `_store_proposals`, the gate has two viable seams, and the choice depends on what “auto-accept” means:

| Seam | File / line | Effect |
|---|---|---|
| **CLI presentation gate** — filter before writing markdown | `reasons/cli.py:2804` (`cmd_propose_update`, right after `api.propose_update` returns) | Skips markdown for `source-divergence`, keeps it for `prior-knowledge`/`detected-contradiction`. Does not touch persistence. |
| **Storage gate** — filter during persistence | `reasons/api.py:4544–4589` (`_store_proposals`, where `basis = p.get("basis", "prior-knowledge")` at 4561) | Skips the `INSERT` (or writes with an `auto_accepted` status) per `basis`. Affects what lands in the `proposals` table. |

The phrasing “skip markdown for `source-divergence`, keep it for `prior-knowledge`” points to the **CLI seam (`reasons/cli.py:2804`)** — that is the point where markdown is generated (`format_proposals_file` imported at 2782) — while the **API seam at `reasons/api.py:4561`** is what you’d touch if “auto-accept” means writing the row through `.store_proposals` without an intervening review step. A robust implementation likely touches both: check `basis` at the storage boundary (4561) to decide auto-commit vs. pending-review, and check it again in `cmd_propose_update` (around 2804–2793) to decide which proposals appear in the emitted `propose-update-*.json` markdown report.

### Alternative entry points
- **MCP** — `reasons/mcp_server.py:268` defines `propose_update(...)` and at **line 287** calls `api.propose_update(...)`. Same upstream (`propose_updates`), different front end. A gate placed only in `cmd_propose_update` will **not** cover MCP clients — put the invariant in `api.propose_update` / `_store_proposals` if you want it enforced everywhere.
- **Library** — `tests/test_propose_update.py:392` calls `propose_updates(...)` directly, proving the library is usable without the CLI.

## Topics to Explore

- [file] `reasons/api.py` — Inspect `propose_update` (line 4458) and `_store_proposals` (around 4544–4589) to decide whether the gate should enforce persistence or just presentation, and to see how `basis` flows from `p.get("basis", "prior-knowledge")` at 4561 into the `INSERT`.
- [file] `reasons/cli.py` — Read `cmd_propose_update` (2775–2810) and the subparser at 3801 to see exactly where the report path, `api.propose_update`, and markdown generation are interleaved.
- [function] `reasons/propose_update.py:parse_update_proposals` — Confirms how unknown `basis` values are coerced to `prior-knowledge`; this determines what a gate sees for malformed LLM output.
- [file] `reasons/mcp_server.py` — The parallel `propose_update` at line 268 shows a second caller of `api.propose_update`; any gate must be mirrored (or pushed down into `api`) to stay consistent.
- [general] `auto-accept-policy-design` — Whether `source-divergence` should bypass the report entirely or just record a status flip; the choice affects whether you gate in `_store_proposals` (row state) vs. `cmd_propose_update` (report output).

## Beliefs

- `cli-to-propose-updates-requires-api-layer` — The `propose-update` CLI command does not call `propose_updates` directly; it goes through `api.propose_update` (imported as `_propose` at `reasons/api.py:4475`).
- `cmd-propose-update-is-the-cli-handler` — `cmd_propose_update` at `reasons/cli.py:2775` is the sole CLI handler for the `propose-update` subcommand, registered in the dispatch dict at `reasons/cli.py:4007`.
- `proposal-basis-defaults-to-prior-knowledge` — Any proposal whose `basis` is missing or invalid is coerced to `prior-knowledge` in `parse_update_proposals` and again at `reasons/api.py:4561`.
- `basis-taxonomy-is-closed` — `propose_updates` only emits bases from `{source-divergence, detected-contradiction, prior-knowledge}` (the `BASES` set in `reasons/propose_update.py`); any gate can exhaustively match on these three values.
- `mcp-propose-update-shares-the-api-path` — `reasons/mcp_server.py:287` calls the same `api.propose_update` as the CLI, so a gate placed only in `cmd_propose_update` will not fire for MCP clients.
