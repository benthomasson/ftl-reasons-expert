---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-27
---

# Topic: Verify whether adding `json5` would be allowed and how its permissive grammar would change parsing contracts.

# Adding `json5`: policy verdict and parsing-contract impact

## Verdict

**Allowed, but the scope matters.**

- Nothing in `CLAUDE.md` forbids new dependencies — its "Design Decisions" section only talks about SQLite, IN/OUT status, BFS propagation, and naming.
- `pyproject.toml` (all 35 lines returned) declares `dependencies = []` and pushes every third-party library into `[project.optional-dependencies]` (`pg`, `cluster`, `mcp`, `api`, `vertex`, `langfuse`, `openai`, `llm`). A `json5` core dependency would break that zero-runtime-dependency invariant; a `json5 = ["json5>=..."]` optional extra would match the existing pattern.
- There are **zero existing mentions of `json5`** (`existing_json5_mentions.total_count == 0`), so no prior design decision to override.

The real work isn't policy; it's deciding **which `json.loads` call sites are external input boundaries** vs. internal serialization round-trips, and auditing the exception handlers. Applying `json5` indiscriminately across all 201 matched sites would be wrong.

---

## Where JSON parsing actually happens

The 201 matches split cleanly into two populations:

### Internal serialization (do **not** switch)

These read back JSON the app itself wrote via `json.dumps`. The format is an internal invariant; loosening it produces no benefit and weakens round-trip guarantees.

| File | Lines | What it parses |
|---|---|---|
| `reasons/storage.py` | migration block in `_init_schema` (parses `metadata_json`, `justifications.antecedents_json`, etc.) | App-written SQLite columns |
| `reasons/api.py` | 5554–5555, 5609–5616, 5660 | App-written row columns (`impact_json`, `snapshot_json`, `result_json`, `tags_json`) |
| `reasons/pg.py` | 371, 416, 631, 665, 694, 708, 710, 772, 829, 883, 972, 974, 983 | App-written Postgres `JSONB`-ish text columns |

Note that `storage.py:save()` writes with `json.dumps(node.metadata)`, `json.dumps(j.antecedents)`, `json.dumps(j.outlist)` — so the read side must stay strict JSON to guarantee "what I wrote is what I read."

### External / untrusted boundaries (candidates for `json5`)

| Site | Likely source | Notes |
|---|---|---|
| `reasons/api.py:2423` `data = json.loads(json_str)` | Probably `import_json` reading a user-supplied file | **The main candidate.** Its body isn't in the observations — see "What's missing" below. |
| `reasons/api.py:4026` `ids = json.loads(match.group())`, `except json.JSONDecodeError` at 4030 | JSON embedded in LLM/agent text | Permissive grammar would help here. |
| `reasons/llm.py:129` `data = json.loads(output)`, `except (json.JSONDecodeError, ValueError)` at 130 | Raw LLM output | Same — comments/trailing commas in model output are common. |
| `reasons/ask.py:122` `obj = json.loads(line)`, `except json.JSONDecodeError` at 125 | Line-delimited JSON stream | Could be a tool channel (internal) or external; unclear. |
| `reasons/api.py:4641`, `4705` `report = _json.load(f)` | File inputs | Aliased import makes source untruthful in grep — verify caller. |
| `reasons/mind_service.py:75` `return json.loads(resp.read().decode("utf-8"))` | HTTP service response | Trusted peer; permissive parsing buys nothing unless that peer is unreliable. |

The JSON import tests (`tests/test_import_json.py`) all construct fixtures with `json.dumps(data)` — strict output. So the *round-trip* contract is "strict in, strict out." Switching the import side to `json5` is additive: existing fixtures still parse, because strict JSON is a subset of JSON5.

But `import_beliefs.py` — which the observation labelled `json_import_implementation` — is actually **markdown parsing** (`parse_beliefs`, `parse_nogoods`, `strip_frontmatter`). It never touches JSON. Flag that the label is misleading; the JSON importer lives in `reasons/api.py`.

---

## How the permissive grammar changes contracts

If you swap `json.loads` → `json5.loads` at the external boundaries above:

### Newly accepted inputs (previously rejected)

- **Comments**: `// line` and `/* block */`.
- **Trailing commas** in objects and arrays.
- **Single-quoted strings**.
- **Unquoted identifier keys**: `{name: "x"}` (identifiers may include `$`, `_`).
- **Number extensions**: leading `+`, hex (`0x1F` → 31), leading-dot (`.5`), `Infinity`, `-Infinity`, `NaN`.
- **Line continuations** inside strings (`\` + newline), and literal multi-line strings.
- **UTF-8 BOM** at file start (Python's strict `json.loads` rejects a BOM; JSON5 permits it) — this is a real, observable behavior change for `import_json`.

### Preserved behaviors

- Duplicate keys: last-wins (same as `json`).
- Strict-JSON inputs still parse identically, so `json.dumps`-produced exports remain importable — the export side does not need to change.

### Data-integrity consequences

- Hex literals and `+` signs are silently normalized to ints/floats. A hand-edited `export.json` with `0x10` becomes `16`, and re-export yields `16` — the file diff is not identity-preserving.
- `Infinity`/`NaN` become Python `float('inf')`/`float('nan')`. Note Python's `json.loads` already accepts these by default (`allow_nan=True`), so this is **not** a delta for Python — but if you later tighten `json` via `parse_constant`, json5 would re-widen it.

### Exception-handling contract changes (the sharp edge)

Three call sites catch `json.JSONDecodeError` specifically:

- `reasons/api.py:4030` → `except json.JSONDecodeError:`
- `reasons/ask.py:125` → `except json.JSONDecodeError:`
- `reasons/llm.py:130` → `except (json.JSONDecodeError, ValueError):`

The PyPI `json5` library raises `ValueError` subclasses (not `json.JSONDecodeError`). That means:

- `llm.py:130` would **continue to work** because it catches `ValueError`.
- `api.py:4030` and `ask.py:125` would **stop catching** malformed input from the json5 parser — errors would propagate as unhandled exceptions to `import_json` callers and CLI users.

So swapping the parser without widening those `except` clauses silently changes error semantics. This is the single most important contract to audit.

---

## Test coverage implications

`tests/test_import_json.py` (193 lines) asserts positive behavior only:

- Round-trips `json.dumps` → `import_json` for premises, justifications, outlist, retracted state, nogoods, metadata, full network.
- `test_missing_file_raises` is the **only** negative test (`FileNotFoundError`).

There is **no** test asserting that malformed JSON raises `JSONDecodeError`, and no test asserting rejection of comments/trailing commas. So adding `json5` acceptance would pass CI silently — which means the contract change is untested. You would want:

- Positive: `import_json` accepts `{/* c */ "nodes": [...],}` and single-quoted strings.
- Negative: genuinely malformed text (`{[`) still raises a catchable error type.
- Boundary: confirm that `import_json("/nonexistent.json")` still raises `FileNotFoundError` (already covered).

---

## What's missing from the observations

1. **The body of `reasons/api.py:import_json`** — we only have the `json.loads(json_str)` line at 2423. Need to confirm the enclosing function is `import_json` and whether `_json` / a helper is used.
2. **Context around `api.py:4020–4035`** — is `match.group()` from regex extraction over LLM text or over a user-supplied IDs string?
3. **Context around `api.py:4635–4710`** — `_json.load(f)` callers; the aliased import means the grep target is non-obvious.
4. **A dependency policy doc other than `pyproject.toml`** — `README.md` and any `CONTRIBUTING.md` are not in the observations. `pyproject.toml` is only 35 lines with no comments about policy, but the empty `dependencies` list is the strongest signal.
5. **The `json5` package's API parity** — whether it supports `parse_float`, `object_pairs_hook`, and BOM handling, and whether its error class hierarchy is stable across versions.
6. **`reasons/api.py:2423`'s file-read path** — whether it opens with `encoding="utf-8"` explicitly; BOM acceptance interacts with this.

---

## Recommendation sketch

1. Keep `dependencies = []`. If adopted, add `json5 = ["json5>=..."]` as an optional extra, mirroring `pg`, `mcp`, `llm`.
2. Apply `json5.loads` only at external boundaries: `api.import_json` (~line 2423) and LLM output parsing (`llm.py:129`). Leave storage, pg, and row-deserialization on strict `json.loads`.
3. Keep `export_network` strict (`json.dumps`) so downstream consumers don't see format drift.
4. Update `api.py:4030` and `ask.py:125` to catch `(json.JSONDecodeError, ValueError)` — matching the pattern already in `llm.py:130`.
5. Add both positive (comments/trailing commas/single quotes) and negative (malformed) tests to `tests/test_import_json.py`.

## Topics to Explore

- [function] `reasons/api.py:import_json` — Confirm the exact parse path and file encoding; line 2423 is the only visible call site.
- [function] `reasons/api.py:_extract_ids` — Context for lines 4020–4035; determines whether the input is model output or user input, and whether the narrow `JSONDecodeError` catch is a real bug.
- [function] `reasons/llm.py:parse_output` — The only handler already tolerant of `ValueError`; the argument for using json5 starts here.
- [general] `json5-library-contract` — Verify exception class hierarchy, BOM handling, and API parity (`parse_float`, `object_pairs_hook`) of the `json5` PyPI package.
- [file] `README.md` — Check whether the zero-runtime-dependency invariant is documented outside `pyproject.toml`.

## Beliefs

- `core-runtime-dependencies-empty` — `pyproject.toml` sets `dependencies = []`; every third-party library is isolated in `[project.optional-dependencies]`.
- `json-parsing-call-sites-are-predominantly-internal` — Of 201 `json.load`/`json.loads` matches, the majority (`reasons/pg.py`, `reasons/storage.py`, `reasons/api.py:5554–5660`) deserialize columns the app itself wrote.
- `json5-mentions-absent` — The repository contains zero occurrences of `json5` (`existing_json5_mentions.total_count == 0`), so no existing code or test asserts json5 semantics.
- `llm-json-handler-catches-valueerror` — `reasons/llm.py:130` catches `(json.JSONDecodeError, ValueError)`, so it would survive a json5 swap; `reasons/api.py:4030` and `reasons/ask.py:125` catch only `JSONDecodeError`.
- `import-json-tests-accept-only-strict-json` — `tests/test_import_json.py` writes fixtures exclusively via `json.dumps` and has no malformed-input test, so json5 acceptance would be unverified by CI.
