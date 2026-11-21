---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-21
---

# Topic: The test suite only exercises `[IN]` and `[STALE]`. Check the parser/importer for handling of `OUT`, `UNKNOWN`, or other statuses before assuming the mapping is complete.

## Explanation

The caution is valid, but scope matters. For `tests/test_import_beliefs.py`, the suite only exercises `[IN]` and `[STALE]`, so the `OUT` path in `reasons/import_beliefs.py` is not covered there. Repo-wide, `tests/test_import_agent.py` does contain some `OUT` coverage, but `UNKNOWN` and other statuses remain unverified and largely unhandled in the markdown parsers.

### Parser behavior: `reasons/import_beliefs.py::parse_beliefs`

The belief-header parser only accepts three statuses:

```python
m = re.match(r"^### (\S+) \[(IN|OUT|STALE)\]\s*(.*)", line)
```

So:

- `[IN]` is parsed.
- `[OUT]` is parsed.
- `[STALE]` is parsed.
- `[UNKNOWN]` and any other status such as `[PROPOSED]` or `[RETRACTED]` are **not** parsed as claim headers.

Worse, the fallback branch has:

```python
elif line.startswith("### "):
    pass  # next claim header handled above
```

So an unrecognized header like `### x [UNKNOWN] OBSERVATION` will fail the header regex and then be silently ignored. If there is an existing `current` claim, subsequent lines under that unknown header may be attached to the previous claim. There is no validation error, no warning, and no `unknown_status` count in the import result. That is the main mapping-completeness risk.

### Importer behavior: `reasons/import_beliefs.py::import_into_network`

The module docstring explicitly maps:

- `IN` with no `depends_on` → premise
- `IN` with `depends_on` → SL-justified node
- `STALE` → OUT node / retracted
- `OUT` → OUT node / retracted

The implementation does:

```python
if claim["status"] in ("STALE", "OUT"):
    network.retract(claim["id"])
    retracted += 1
```

So `OUT` **is handled** in `import_into_network`, but only as “add the node, then retract it.” It is not distinguished from `STALE` in `truth_value`; both end up `OUT`. The `retracted` count combines both. There is no branch for `UNKNOWN` or any other status.

### Test coverage: `tests/test_import_beliefs.py`

The `SAMPLE_BELIEFS` fixture contains only:

- `premise-a [IN]`
- `premise-b [IN]`
- `derived-c [IN]`
- `stale-d [STALE]`
- `chain-e [IN]`

The status assertions only check `IN` and `STALE`:

```python
assert by_id["premise-a"]["status"] == "IN"
assert by_id["stale-d"]["status"] == "STALE"
```

And the retraction count expects exactly one retracted claim, `stale-d`. So the `OUT` branch in `import_into_network` is **not exercised by this test module**.

### Repo-wide, some `OUT` coverage exists

`tests/test_import_agent.py` does include `OUT` cases, for example:

- `test_import_agent_preserves_out_status`
- `test_import_agent_supersession_preserved`
- `test_import_agent_json`, where `blocker-c` has `"truth_value": "OUT"`

`tests/test_ask.py:129` also allows output lines starting with `[IN]` or `[OUT]`.

So the blanket statement “the test suite only exercises `[IN]` and `[STALE]`” is too broad repo-wide, but it is accurate for `tests/test_import_beliefs.py`, and `UNKNOWN` is not tested anywhere visible in the provided observations.

### Agent importer: `reasons/import_agent.py`

For markdown, `import_agent` reuses `parse_beliefs`, so it inherits the same limitation: only `IN`, `OUT`, and `STALE` can appear as parsed statuses. Its normalization says:

```python
is_out = c["status"] in ("STALE", "OUT")
```

So `OUT` is treated as out, and `UNKNOWN` cannot reach this code through the markdown parser.

For JSON, `_normalize_json` uses:

```python
is_out = ndata.get("truth_value") == "OUT"
```

That means a JSON node with `"truth_value": "UNKNOWN"` would enter the non-out branch. The provided snippet does not show an `UNKNOWN`-specific retraction or rejection path. Also, `only_in=True` filters strictly to `truth_value == "IN"`, so `UNKNOWN` would be excluded in that mode.

### `UNKNOWN` elsewhere is not a parser mapping

`UNKNOWN` appears in `reasons/api.py:3764` and `reasons/api.py:3836` as a fallback `truth_value` for `premises_data`:

```python
"truth_value": "UNKNOWN"
```

It also appears in `reasons/report_belief.py:38`, `:85`, `:123`, `:149`, and `:194` as a fallback when node detail lacks `truth_value`:

```python
status = node_detail.get("truth_value", "UNKNOWN")
```

Those are display/report defaults, not evidence that the beliefs.md parser or importer supports `[UNKNOWN]`.

### Conclusion

- `OUT` **is parsed** by `parse_beliefs`.
- `OUT` **is mapped to retraction** in `import_into_network`.
- `OUT` is **not tested in `tests/test_import_beliefs.py`**, though it is tested elsewhere via `import_agent`.
- `UNKNOWN` is **not parsed** from markdown claim headers.
- Other statuses are silently ignored by `parse_beliefs`.
- `import_agent` JSON normalization only checks for exactly `"OUT"`, so `UNKNOWN` JSON truth values have no visible special handling.

The provided observations do not include the remainder of `_import_claims` in `reasons/import_agent.py` or the network recomputation internals, so the exact final `truth_value` for an `UNKNOWN` JSON import is not fully proven here. But the parser/normalizer clearly has no `UNKNOWN` branch.

## Topics to Explore

- [file] `reasons/import_beliefs.py` — Inspect `parse_beliefs` and `import_into_network` to confirm the `IN`/`OUT`/`STALE` regex and the combined retraction path.
- [file] `tests/test_import_beliefs.py` — Add or inspect cases for `[OUT]` and `[UNKNOWN]`; the current fixture only covers `IN` and `STALE`.
- [file] `reasons/import_agent.py` — Compare `_normalize_markdown` and `_normalize_json`; JSON only treats exact `"OUT"` as out.
- [function] `reasons/import_beliefs.py:parse_beliefs` — Verify how unrecognized `### id [STATUS]` lines are silently ignored.
- [general] `unknown-status-policy` — Decide whether `UNKNOWN` should be rejected, parsed as a claim, or mapped to `OUT`.

## Beliefs

- `import-beliefs-parser-recognizes-only-in-out-stale` — `parse_beliefs` accepts only `IN`, `OUT`, and `STALE` in claim header regexes.
- `import-beliefs-maps-stale-and-out-to-retraction` — `import_into_network` retracts nodes whose parsed status is either `STALE` or `OUT`.
- `import-beliefs-tests-lack-out-coverage` — The `tests/test_import_beliefs.py` fixture contains only `IN` and `STALE` claims, with no `OUT` header.
- `import-agent-json-normalization-only-checks-out` — `_normalize_json` marks a node out only when `truth_value == "OUT"`, so `UNKNOWN` is not treated as out.
- `unknown-markdown-status-is-silently-ignored` — A `### id [UNKNOWN] ...` header fails `parse_beliefs` and is ignored rather than creating a claim.
