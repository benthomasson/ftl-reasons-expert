---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: The PG path forwards pre-normalized `claims`/`nogoods` while SQLite forwards raw text; differences in behavior here (e.g., `nogoods_file` being ignored for JSON) are a real parity gap worth auditing.

## What the parity gap is about

The short version: **the two backends split responsibility for parsing at different layers**, so the shape of what they receive at the persistence boundary is different — and any parameter that assumes the "raw text" shape (like `nogoods_file`) can silently behave differently across backends.

### SQLite path: parsing happens inside `import_into_network`

On the SQLite side, `reasons/import_beliefs.py` owns normalization entirely:

- `parse_beliefs(text)` (around lines 79–153) turns raw markdown into a list of `claims` dicts.
- `parse_nogoods(text)` (around lines 156–203) turns raw nogoods markdown into a list of nogoods dicts.
- `import_into_network(network, beliefs_text, nogoods_text)` (lines ~206–299) accepts **raw text for both** and does the parsing, topological sort, node creation, and nogood insertion in one shot.

`reasons/storage.py` then persists whatever objects ended up on `network.nogoods` — `Storage.save()` serializes `nogood.nodes` via `json.dumps(...)`, and `Storage.load()` deserializes via `json.loads(...)` (see the `nogoods` handling in `save()` / `load()`). Storage itself never sees raw markdown.

So on SQLite, `nogoods_file` is just a path the API layer reads into a `nogoods_text` string that is then merged/parsed by `import_into_network`. The merge point is unambiguous.

### PG path: pre-normalized pieces are forwarded, and the shape varies per method

The dispatch tests in `tests/test_pg_dispatch.py` reveal the contract the PG backend actually receives:

- `TestImportBeliefsDispatch::test_dispatches_text` asserts `PgApi.import_beliefs` receives `beliefs_text` containing raw markdown ("`assert "alpha" in call_kwargs["beliefs_text"].lower()`"). So **`import_beliefs` forwards raw text on PG too**.
- `TestImportAgentDispatch::test_dispatches_markdown` asserts `PgApi.import_agent` receives ``call_kwargs["claims"]`` with `len == 1`. That is a **pre-normalized list**, not the raw beliefs file.
- `TestImportAgentDispatch::test_dispatches_json` only asserts `call_kwargs["agent_name"] == "remote"` — it never asserts what happens to `claims`, `nogoods`, or a `nogoods_file`-derived payload on the JSON branch.
- `TestImportJsonDispatch::test_dispatches_parsed_data` asserts `PgApi.import_json` receives `data=data` — a **pre-parsed JSON dict** — via `assert_called_once_with(data=data)`.
- `TestSyncAgentDispatch::test_dispatches_markdown` asserts nothing about `claims`/`nogoods` kwargs.

So even *within the PG path* the incoming shapes differ:

| Public API | PG receives |
|---|---|
| `import_json` | parsed JSON dict (`data=...`) |
| `import_beliefs` | raw markdown string (`beliefs_text=...`) |
| `import_agent` (md) | normalized `claims=[...]` |
| `import_agent` (json) | normalized claims/nogoods presumably, but assertions don't cover it |

### Where `nogoods_file` is handled (and why it's fragile)

`reasons/api.py` is where the split lives:

- `import_beliefs` signature at `reasons/api.py:1890` and its one `if nogoods_file:` block at **1905–1908** (raises `FileNotFoundError` if missing, then reads it).
- `import_agent` signature at `:1927`, with **two** `if nogoods_file:` blocks — **1960–1963** and **1995–1998**. That maps to the two input branches (markdown vs JSON).
- `sync_agent` signature at `:2019`, again with two blocks — **2048–2051** and **2083–2086**.
- CLI call sites: `reasons/cli.py:895`, `:925`, `:956` forward `args.nogoods_file` to those three API functions; `:3433`, `:3440`, `:3446` register `--nogoods`.

The critical detail: `import_agent` and `sync_agent` both have **two** file-reading branches, but only the tests for `import_agent(markdown)` show `claims` being forwarded as a list. For `import_agent(json)`, the assertion only checks `agent_name`. That's exactly where the parity gap lives: if the JSON branch forwards `claims`/`nogoods` from the parsed JSON and the markdown branch forwards `claims`/`nogoods` from parsing, then a user who *also* passes `--nogoods nogoods.md` alongside a `network.json` can end up in a path where `nogoods_file` is read (the file must exist, hence the `FileNotFoundError` at 1995–1998 / 2083–2086) but **the parsed nogoods are never merged in**, because the downstream `PgApi.import_agent` / `sync_agent` receives `claims` from JSON instead of `nogoods_text`.

On the SQLite path, the counterpart is `import_into_network(network, beliefs_text, nogoods_text)` — it always consumes `nogoods_text` and always merges it into `network.nogoods`. So JSON imports on SQLite, if they go through a different route (`import_json`), might or might not have the same blind spot — **that's the audit target.** The observation that `nogoods_file` "is being ignored for JSON" is only consistent with one backend if the other reads it; the tests don't pin down which one.

### Why this matters in practice

1. **Silent drops**: `reasons/api.py:1905–1908`, `1960–1963`, `1995–1998`, `2048–2051`, `2083–2086` all enforce file-existence with `FileNotFoundError`. So a file that is "ignored for JSON" still has to physically exist — a footgun that hides the semantic no-op behind an I/O precondition.
2. **Divergent data**: `Nogood` objects are serialized identically by `Storage.save()` (`json.dumps(nogood.nodes)`) and presumably by the PG `rms_nogoods.nodes JSONB` column, so the *stored* representation is equivalent. The divergence is entirely in **which nogoods ever reach the writer**. A JSON import + `--nogoods extra.md` will produce different network contents on SQLite vs PG depending on the branch taken.
3. **Test blind spot**: `TestImportAgentDispatch::test_dispatches_json` and `TestSyncAgentDispatch::test_dispatches_markdown` don't assert on `claims`/`nogoods`/`nogoods_text`, so the parity gap is untested. Any audit should first add assertions there.

### Data-quality caveats in the observations

Two `locate_*` observations are inconsistent with the code that was returned:

- `locate_claims_handling` reports 0 matches for `claims`, but `reasons/import_beliefs.py:parse_beliefs` (returned in `read_import_beliefs_path`) has `claims = parse_beliefs(beliefs_text)` and `tests/test_pg_dispatch.py` has `call_kwargs["claims"]` and `result["claims_imported"]`. The locate result is almost certainly scoped incorrectly — don't trust its 0-count.
- `find_nogoods_related_tests` reports 0 matches, but `tests/test_pg_dispatch.py` contains `test_contains_nogoods` and `"nogoods_imported": 0` in multiple `MagicMock` returns. Again, the observation is incomplete.

### What's still missing

To close the audit you'd need:

- The **body** of `reasons/api.py:import_beliefs`, `:import_agent`, `:sync_agent` between roughly 1890 and 2100 — specifically what each `if nogoods_file:` block *does* after the `FileNotFoundError` check (does it call `parse_nogoods` and pass `nogoods=[...]`? does it pass `nogoods_text`? does it just merge into a local dict?).
- The **`PgApi.import_agent` / `PgApi.sync_agent` / `PgApi.import_json` / `PgApi.import_beliefs`** methods in `reasons/pg.py` — they're past the 800-line slice returned here. Without them we can't say whether PG merges any `nogoods` kwarg at all, or only trusts `claims`.
- The **SQLite `import_json`** implementation in `reasons/api.py` (not in scope here) — to compare whether it also ignores `nogoods_file` for JSON.
- The actual bodies of `reasons/api.py:import_beliefs` at 1890 vs `:import_agent` at 1927 — since `import_beliefs` reads `beliefs_text` and one nogoods file, while `import_agent` reads **two** branches, it's plausible the `import_beliefs` path is already correct and only `import_agent`/`sync_agent` have the JSON blind spot.

## Topics to Explore

- [function] `reasons/api.py:import_agent` — Has two `if nogoods_file:` branches (lines 1960–1963 and 1995–1998); the audit hinges on what the JSON branch actually does with the parsed nogoods
- [function] `reasons/api.py:sync_agent` — Same dual-branch structure at 2048–2051 and 2083–2086; likely shares the `import_agent` gap
- [function] `reasons/pg.py:PgApi.import_agent` — Not in the returned slice; needed to confirm whether PG accepts `nogoods=` at all or ignores `nogoods_file` for JSON
- [function] `reasons/import_beliefs.py:import_into_network` — The SQLite normalization point; compare its `(beliefs_text, nogoods_text)` contract against what PG's `import_agent` receives
- [general] `json-import-parity` — End-to-end compare of SQLite `import_json` vs `PgApi.import_json(data=...)`, tracking whether `--nogoods` is merged on both sides or silently dropped

## Beliefs

- `import-agent-forwards-pre-normalized-claims` — `import_agent` passes a parsed `claims` list to `PgApi.import_agent`, per `tests/test_pg_dispatch.py::TestImportAgentDispatch::test_dispatches_markdown`
- `import-beliefs-forwards-raw-text` — `import_beliefs` passes raw `beliefs_text` to `PgApi.import_beliefs`, per `tests/test_pg_dispatch.py::TestImportBeliefsDispatch::test_dispatches_text`
- `import-json-forwards-parsed-data` — `import_json` passes an already-parsed `data` dict to `PgApi.import_json`, per `test_dispatches_parsed_data` and `mock_pg.import_json.assert_called_once_with(data=data)`
- `sqlite-parsing-lives-in-import-beliefs` — On the SQLite path, all text parsing happens in `reasons/import_beliefs.py:parse_beliefs` / `parse_nogoods` / `import_into_network`; `Storage.save()`/`load()` only serialize/deserialize `Nogood.nodes` via JSON
- `two-nogoods-file-branches-per-json-capable-importer` — `import_agent` and `sync_agent` each contain two distinct `if nogoods_file:` blocks in `reasons/api.py` (1960/1995 and 2048/2083), corresponding to the markdown and JSON input branches
- `nogoods-file-read-requires-file-existence` — Every `if nogoods_file:` block in `reasons/api.py` (1905–1908, 1960–1963, 1995–1998, 2048–2051, 2083–2086) raises `FileNotFoundError` when the path does not exist, even if the downstream branch might not use the parsed result
- `json-pg-dispatch-assertions-incomplete` — `TestImportAgentDispatch::test_dispatches_json` and `TestSyncAgentDispatch::test_dispatches_markdown` do not assert on `claims`/`nogoods` kwargs, so the JSON-side parity behavior is currently untested
