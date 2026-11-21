---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: Cross-check each `Usage` transcript against `tests/test_cli.py` golden output, verify the "211 tests" claim, and flag the ~13 modules with zero README coverage.

# Cross-check: README Usage vs `tests/test_cli.py` — plus the "211 tests" and module-coverage claims

The observations give us the full `README.md` (198 lines returned of 199), the full `tests/test_cli.py` (~1616 lines of 1617), a search result for `"211"` (single hit at `README.md:194`), a module-name regex over the README, and a **broken** `count_test_functions` result (`total_count: 0`). Here is what can and cannot be concluded.

---

## 1. Cross-checking `## Usage` (`README.md:35`) against `tests/test_cli.py`

### 1a. Where they agree

These Usage examples / Commands-table entries have matching assertions in `test_cli.py`:

| README usage | Test that verifies it |
|---|---|
| `reasons init` | `TestInit::test_init_creates_db` asserts `"Initialized"`; `test_init_refuses_existing` asserts `"--force"` |
| `reasons add ID TEXT` | `TestAdd::test_add_premise` asserts `"Added a [IN]"` and `"premise"` in lowercased out |
| `reasons add ID --sl a` | `TestAdd::test_add_derived` asserts `"Added b [IN]"` |
| `reasons add ID --unless y` (outlist) | Indirectly via `TestExplainBranches::test_explain_with_outlist` / `test_explain_violated_outlist` (`"unless:"` / `"violated unless:"`) |
| `reasons status` | `test_status_with_nodes` asserts `"2 IN"`, `"Premises: 1"`, `"Derived: 1"`; `test_status_all` asserts `"[+] a"`, `"[+] b"`; `test_status_empty` asserts `"No nodes"` |
| `reasons show ID` | `test_show_premise` asserts `"ID:     a"`, `"Text:   Premise A"`, `"Status: IN"`, `"Premise (no justifications)"`; `test_show_derived` asserts `"SL(a)"`; `test_show_dependents` asserts `"Dependents: b"`; `test_show_retract_reason` asserts `"Retract reason: ..."` |
| `reasons explain ID` | `test_explain_derived` (`"antecedents: a"`), `test_explain_with_label` (`"test-label"`), `test_explain_with_outlist` (`"unless:"`), `test_explain_failed_antecedent` (`"failed:"`), `test_explain_violated_outlist` (`"violated unless:"`) |
| `reasons trace ID` | `test_trace_derived` asserts `"1 premise"` and `"[+] a"`; `test_trace_names_only` asserts `"rests on"` is absent in names-only mode |
| `reasons nogood A B` | `test_nogood` asserts `"Recorded"`, `"a"`, `"b"` |
| `reasons retract ID` / `reasons assert ID` (single node) | `test_retract` asserts `"Retracted a"`; `test_assert` asserts `"Asserted a"`; already-OUT / already-IN variants assert `"already OUT"` / `"already IN"` |
| `reasons export` | `test_export_json` parses stdout as JSON and checks `data["nodes"]` contains `"a"` |
| `reasons export-markdown -o FILE` | `test_export_markdown_to_file` asserts `"Written to"` and reads file back |
| `reasons hash-sources` | `test_hash_sources_backfill` (`"backfilled"`), `test_hash_sources_force` (`"re-hashed"`), `test_hash_sources_nothing_to_hash` (`"No nodes to hash"`) |
| `reasons check-stale` | `test_check_stale_all_fresh` (`"fresh"`), `test_check_stale_detects_change` (`"STALE"` + exit 1), `test_check_stale_detects_deleted` (`"DELETED"` + exit 1) |
| `reasons log` | `test_log_empty` (`"No propagation events"`), `test_log_with_entries` (`"add"`, `"retract"`) |
| `reasons import-beliefs FILE` | `test_import_beliefs` asserts `"Imported 3 claims"` |
| `reasons challenge` / `reasons defend` | `test_challenge_and_defend` asserts `"Challenged claim"`, `"challenge-claim"`, `"Defended claim"` |

### 1b. Where they disagree — the README is stale

The clearest mismatch is the **cascade format** in the `retract` / `assert` / `nogood` transcripts:

- `README.md` (Usage, around the `retract` example):
  ```
  reasons retract source-uses-langgraph
  # Retracted: source-uses-langgraph, topology-is-static, no-runtime-modification
  ```
  and `reasons assert source-uses-langgraph` → `# Asserted: source-uses-langgraph, topology-is-static, ...`

- `tests/test_cli.py::TestRetractAssert::test_retract` asserts `"Retracted a"` (single id), while `test_retract_cascade` asserts `"Went OUT"` and `"b"` (cascade reported separately). Same split on `assert` → `"Went IN"` in `test_assert_cascade`. The actual output is therefore `Retracted <id>` followed by a distinct `Went OUT: ...` line — **not** a comma-separated `Retracted: a, b, c`.

- The same stale pattern repeats for `reasons retract beliefs-improve-accuracy` (post-import transcript: `Retracted: beliefs-improve-accuracy, engineering-intuition-unreliable, ...`) and for the `nogood` transcript line `Retracted: graph-is-dynamic` — no test asserts a comma-list.

### 1c. Where README format is plausible but not directly asserted

These README formats are **not** confirmed by the tests — only weaker tokens are asserted:

| README transcript | What the test actually checks | Gap |
|---|---|---|
| `status`: `[+] graph-has-cycles: Graph contains cycles  (premise)` + `4/4 IN` | `"[+] a"` (default `--all` only) and `"2 IN"` count token | Per-node text format and the `N/N IN` ratio line are unverified |
| `search "tool-use"`: `(5 dependents)` and `7 results` | `"a" in out`; names-only asserts bare IDs | `(N dependents)` suffix and `N results` summary unverified |
| `check-stale`: `5 fresh, 14 STALE (of 19 checked)` | `"fresh"`, `"STALE"`, `"DELETED"` tokens | Summary line format unverified |
| `import-beliefs`: `Imported 39 claims (2 retracted)` | `"Imported 3 claims"` | `(N retracted)` parenthetical unverified |
| `nogood`: `Recorded nogood-001: ...` | `"Recorded"` and both ids | `nogood-NNN` numbering scheme unverified |
| `challenge`/`defend`: `Changed: velocity-constraint, acoustic-metric-schwarzschild, ...` | Only `"Challenged <id>"` prefix | `Changed:` line unverified |
| `explain`: `SL justification valid — antecedents: topology-is-static` | `"antecedents:"` substring | Em-dash phrasing unverified |

### 1d. README-documented features that `test_cli.py` does **not** test

These are explicitly claimed in the Usage block or Commands table but have zero corresponding assertions:

- `reasons add ID "text" --cp a,b` (`README.md:170`) — no `--cp` anywhere in `test_cli.py`.
- `reasons import-beliefs FILE --nogoods FILE` — tests rely on auto-detected `nogoods.md` or inline `## Nogoods` sections (`test_import_beliefs_with_nogoods`); the explicit flag is untested.
- `reasons list --has-dependents` — no test uses it (only `--premises`, `--status`, and default are covered).
- `reasons list --challenged` (`README.md:159`) — no test uses it.
- `reasons compact --budget N` — `test_compact` only asserts `"a" in out`; `--budget` is never passed even though the README uses `--budget 500` (`README.md:128`).
- `reasons add ... --sl a,b,c` tip text: the README doesn't mention this, but `test_add_multi_premise_tip` asserts `"Tip"` and `"--any"` appear — both undocumented in the README.

### 1e. The reverse gap: tested commands not in the README at all

`test_cli.py` exercises a large surface area that the README Usage / Commands table does not document:

`what-if`, `supersede`, `check-integrity`, `backfill-hashes`, `convert-to-premise`, `add-repo` / `repos`, `import-json`, `summarize`, `import-agent` / `sync-agent` / `namespaces`, `deduplicate` (including `--auto`, `--semantic`, `--accept`, `--output`), `derive --dry-run`, `accept`, `lookup`, `trace-access-tags`, the entire proposal lifecycle (`propose-retract`, `propose-supersede`, `propose-add`, `proposals`, `proposal`, `accept-proposal`, `reject-proposal`, `withdraw-proposal`, `propose-nogood`, `propose-update`), the `--format names-only` option on `status`/`search`/`list`/`compact`/`trace`/`explain`, and the `--access-tags` / `--visible-to` privacy flags.

That is ≈20 undocumented CLI commands plus ≈10 undocumented flags — substantially more missing than the README's "Commands" table (`README.md:163-187`) suggests.

---

## 2. Verifying the `"211 tests"` claim

- The only occurrence of `"211"` in the repo is `README.md:194` (from `find_211_tests_claim`, `total_count: 1`). It reads:
  > 211 tests covering propagation, retraction cascades, restoration, multiple justifications, diamond dependencies, nogoods, dependency-directed backtracking, non-monotonic justifications (outlist), dialectical argumentation (challenge/defend), explain traces, SQLite round-trips, beliefs.md import, export-markdown, check-stale, hash-sources, compact, search, and list.

- `count_test_functions` returned `"total_count": 0` — **this is a tool failure**, not a fact. Manually counting `def test_*` in `tests/test_cli.py` (by test class) gives roughly **151** test methods. The tool clearly searched the wrong path or the `matches` array was dropped. Do not trust the `0`.

- `find_parametrized_tests` returned `0` — this is consistent with the file: only `@pytest.mark.skipif` appears (on `TestDeduplicateSemantic`), and no `@pytest.mark.parametrize` is visible. So the "211" figure is not parametrize expansion; it must be a raw count of test functions.

- Conclusion: **211 cannot come from `test_cli.py` alone** (~151 here). The README's own Tests section (`README.md:~190`) invokes `pytest tests/ -v`, i.e. the whole directory. Several listed coverage topics — "*multiple justifications, diamond dependencies, dependency-directed backtracking*" — have **no** representative test in `test_cli.py`, which strongly implies the remaining ~60 tests live in non-CLI test files (e.g. `tests/test_tms.py`, `tests/test_network.py`, `tests/test_storage.py`). None of those files are in the observations.

**Verdict on the claim:** unverifiable from the provided data. To settle it (`pytest --collect-only -q tests/` is the authoritative check) we still need:
1. the directory listing of `tests/` and per-file counts,
2. whether `TestDeduplicateSemantic`'s two `skipif`-gated tests run in the release environment (they add up to 2 to the total either way, but affects the "runnable" vs "collected" reading).

---

## 3. Flagging the modules with zero README coverage

### 3a. What the observation actually shows

The `find_module_mentions_in_readme` regex contains 34 candidate names:
`api, ask, build_wiki, check_stale, cli, cluster, compact, contradictions, derive, export_card, export_markdown, hf, import_agent, import_beliefs, llm, mcp_client, mcp_server, merkle, metadata, mind_service, network, pg, propose_update, pubsub, repair, repair_premises, report_belief, report_gated, review, review_justifications, review_premises, storage, verify`

Only **6 README lines** matched, and **only 2 module names** actually matched (the rest are `reasons.`):
- `network` — `README.md:3` ("dependency network") and `:183`
- `compact` — `README.md:128`, `:186`, `:194`

That is far below the "~13 uncovered modules" claim, but the count in the observations is misleading for two reasons:

1. **Hyphen vs. underscore.** README documents the CLI as `check-stale`, `export-markdown`, `import-beliefs` (hyphenated), while the regex looks for `check_stale`, `export_markdown`, `import_beliefs`. Those three are conceptually documented but would be flagged by a naive module-name search.
2. **No `src/reasons/` inventory was included.** We can't know which of the 34 candidates actually correspond to a `.py` file in this repo versus being CLI subcommand words or infrastructure.

### 3b. Modules with **strictly zero** README references (from the regex)

Excluding the three hyphenated-but-covered ones (`check_stale`, `export_markdown`, `import_beliefs`) and the two regex matches (`network`, `compact`), the un-mentioned candidates are **≈29**, e.g.:

`api, ask, build_wiki, cli, cluster, contradictions, derive, export_card, hf, import_agent, llm, mcp_client, mcp_server, merkle, metadata, mind_service, pg, propose_update, pubsub, repair, repair_premises, report_belief, report_gated, review, review_justifications, review_premises, storage, verify`

### 3c. To land precisely on "~13"

Which 13 the author has in mind almost certainly depends on filtering "public feature module" vs "infrastructure". Plausible groupings:

- **CLI + test-dependent modules with no README mention:** `cli`, `storage`, `cluster`, `derive`, `propose_update`, `import_agent`, `contradictions`, `export_card` — note that `reasons.storage` is imported directly inside `test_cli.py` (`TestCheckIntegrity`, `TestBackfillHashes`, `TestPropagateWithChanges`), so a module can be **exercised by tests yet still have zero README coverage** — this is exactly the situation the README should fix.
- **Proposal / review subsystem (completely undocumented):** `review`, `review_justifications`, `review_premises`, `propose_update`, `report_belief`, `report_gated`.
- **Infrastructure / integrations (arguably out of README scope):** `pg`, `pubsub`, `merkle`, `metadata`, `verify`, `mcp_client`, `mcp_server`, `mind_service`, `hf`, `llm`.
- **"Wiki/export" helpers:** `build_wiki`, `export_card`.

**Missing data to confirm exactly 13:** the `src/reasons/` listing (or `pyproject.toml` `packages`), plus a per-module `grep` against the README using both underscore and hyphen variants.

---

## Topics to Explore

- [file] `tests/test_cli.py` — Re-read `TestRetractAssert` and `TestNogood` to decide whether to fix the README (`Retracted: a, b, c` → `Retracted <id>` + `Went OUT: ...`) or change the CLI to match the documented format.
- [file] `tests/test_cli.py` — Audit `TestProposalLifecycle`, `TestImportAgent`, `TestDeduplicate`, `TestNamesOnlyFormat`, and `TestTraceAccessTags` to enumerate the ~20 commands and ~10 flags that should be added to `README.md`'s Commands table.
- [function] `tests/test_cli.py:TestAddJustification::test_add_justification` — Confirms `add-justification` (which the README doesn't document) is tested, confirming the README undercounts the CLI surface.
- [general] `tests-directory-inventory` — Run `pytest --collect-only -q tests/` and count per file; the "211 tests" claim is only meaningful as a whole-`tests/` number, not a `test_cli.py` number.
- [general] `src-reasons-module-inventory` — List `src/reasons/*.py` and grep README for each in underscore and hyphen form to pin down the exact 13 uncovered modules.

---

## Beliefs

- `readme-cascade-format-contradicts-tests` — `README.md` shows `Retracted: a, b, c` / `Asserted: a, b, c`, but `tests/test_cli.py::TestRetractAssert::test_retract_cascade` asserts `"Retracted <id>"` and `"Went OUT"` as separate lines, so the README transcript is stale.
- `211-test-claim-is-whole-suite-not-cli-only` — `tests/test_cli.py` contains ~151 `def test_*` methods, and README topics like *diamond dependencies* and *dependency-directed backtracking* have no representative test in `test_cli.py`, so `README.md:194`'s "211" must include non-CLI test files.
- `count-test-functions-observation-is-wrong` — `count_test_functions` returned `total_count: 0` despite ~151 visible `def test_*` in `tests/test_cli.py`, so the tool searched a wrong path and cannot be used to verify the 211 figure.
- `cli-tests-cover-commands-readme-omits` — `tests/test_cli.py` exercises `what-if`, `supersede`, `check-integrity`, `backfill-hashes`, `convert-to-premise`, `add-repo`/`repos`, `import-json`, `summarize`, `import-agent`/`sync-agent`/`namespaces`, `deduplicate`, `derive`, `accept`, `lookup`, `trace-access-tags`, the full `propose-*`/`accept-proposal`/`reject-proposal`/`withdraw-proposal`/`propose-nogood`/`propose-update` family, and `--format names-only`, none of which appear in `README.md`.
- `readme-flags-untested-in-cli` — `add --cp`, `import-beliefs --nogoods`, `compact --budget`, `list --has-dependents`, and `list --challenged` are documented in `README.md` but never exercised by `tests/test_cli.py`.
- `readme-hyphen-command-names-evade-underscore-search` — README documents `check-stale`, `export-markdown`, and `import-beliefs` (hyphens), so an underscore-module regex reports them as uncovered even though they're command-documented; any "zero coverage" count must treat these as covered.
- `storage-module-is-tested-but-undocumented` — `reasons.storage.Storage` is imported directly inside `tests/test_cli.py` (`TestCheckIntegrity`, `TestBackfillHashes`, `TestPropagateWithChanges`) yet has no README mention, proving tests can exercise a feature the README never surfaces.
