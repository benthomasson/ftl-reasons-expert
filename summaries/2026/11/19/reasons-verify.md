---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-19
---

# File: reasons/verify.py

# `reasons/verify.py` — Belief-to-Source Verification

## Purpose

This module owns the **rigorous, source-grounded verification** of beliefs. Given a belief (a claim plus a source reference) and the *actual text* of the source document, it asks an LLM to decide whether the claim is **CONFIRMED / STALE / PARTIAL / INCONCLUSIVE**, and to return a verbatim quote that justifies the verdict.

The docstring explicitly contrasts this with `report_belief.py`: verification asks *"does the source literally say this?"*, whereas reporting asks *"synthesize a narrative about this belief."* It is a checkpoint, not a summarizer — the design intent is adversarial against plausible-sounding confabulation.

It is **not** responsible for locating beliefs or orchestrating a batch run; it exposes small composable helpers plus one LLM call, and lets the caller (`api.py`) wire them together.

---

## Key Components

### Constants

- **`MAX_SOURCE_CHARS = 30_000`** — hard cap on the amount of source text fed to the model. Prevents runaway prompts/context blow-ups on large documents.
- **`VERIFY_PROMPT`** — a `str.format` template. Defines the four verdict classes, mandates JSON-only output, requires a *verbatim* quote, and forces `INCONCLUSIVE` when the source is unavailable. Note the doubled braces (`{{ … }}`) around the embedded JSON example so `.format()` doesn't choke on them.

### Functions

1. **`read_source(source, db_path=None) -> str | None`**
   - Delegates path resolution to `check_stale.resolve_source_path`, using the directory containing the DB (`Path(db_path).parent`) as the base for relative paths.
   - Returns `None` if the path can't be resolved, doesn't exist, or `read_text` fails.
   - Truncates at `MAX_SOURCE_CHARS` and appends `"\n\n[... truncated ...]"` so the LLM knows the text is incomplete.
   - **Contract**: never raises; either returns a string or `None`.

2. **`format_belief_for_verify(belief, source_content) -> str`**
   - Renders an `### <id>` heading, `**Claim:**`, optional `**Source file:**` and `**Source URL:**`, then a fenced block containing `source_content` (or the literal `(source not available)`).
   - Uses `belief["id"]`/`belief["text"]` as required keys; `source` and `source_url` are optional (`belief.get(...)`).
   - `source_content or "(source not available)"` means *both* `None` and empty-string collapase to the placeholder.

3. **`parse_verify_response(response) -> dict`**
   - Extracts the first `{...}` blob via `re.search(r"\{.*\}", response, re.DOTALL)` (greedy — see caveats in Invariants).
   - `json.loads` the match; on `JSONDecodeError`/`TypeError` returns `{}`.
   - Iterates `data.items()` and **drops** entries that aren't dicts, lack `"verdict"`, or whose `"verdict"` isn't a `str`.
   - Uppercases the verdict (so `"confirmed"` → `"CONFIRMED"`) and defaults `reason` to `""`, `quote` to `None`.
   - **Contract**: never raises.

4. **`verify_beliefs(beliefs_with_sources, model="claude", timeout=120) -> dict`**
   - The only function that talks to the LLM.
   - Builds the belief section by joining each `format_belief_for_verify(b, src)` with `"\n\n---\n\n"`.
   - Lazy-imports `invoke_model`, formats the prompt, calls the model, and feeds the response to `parse_verify_response`.

---

## Patterns

- **Function-local imports** for `resolve_source_path` and `invoke_model`. This is deliberate: `check_stale` and `llm` are heavier and/or more deeply linked into the package; deferring the import breaks what would otherwise be a circular import (e.g., `api.py` → `verify.py` → `check_stale.py` → … → `api.py`).
- **Layered responsibilities**: `read_source` handles I/O; `format_belief_for_verify` handles prompt rendering; `parse_verify_response` handles parsing; `verify_beliefs` handles the LLM round-trip. The caller composes them per belief — verify_beliefs never touches disk.
- **Fail-soft parsing**: every failure mode (missing JSON, bad structure, non-string verdict) returns an empty/partial dict rather than an exception. Callers must treat missing IDs as "unknown."
- **Regex-scavenged JSON** — a routine pattern in LLM integrations to tolerate markdown fences and preamble chatter around the JSON payload.
- **Prompt as a `str.format` template** with `{beliefs}` slot — allows tests to substitute canned payloads without invoking a model.

---

## Dependencies

**Imports (out)**:
- Standard library: `json`, `re`, `pathlib.Path`.
- `reasons.check_stale.resolve_source_path` — deferred; owns the mapping from `source` string to a filesystem `Path`, including relative-to-DB resolution.
- `reasons.llm.invoke_model` — deferred; the provider abstraction that actually calls Claude (or whatever `model` names).

**Imported by (in)**:
- `reasons/api.py` — the public API layer that orchestrates verification over beliefs; supplies `beliefs_with_sources`.
- `reasons_lib/verify.py` — a mirrored copy (the repo appears to ship the module under two package names).
- `tests/test_verify.py` — tests the parsing, formatting, and prompt assembly, presumably with mocked `invoke_model`.

---

## Flow

A typical invocation from `api.py` looks roughly like:

```
beliefs_with_sources = []
for belief in selected_beliefs:
    src = read_source(belief["source"], db_path=...)   # optional; None if missing
    beliefs_with_sources.append((belief, src))

results = verify_beliefs(beliefs_with_sources, model=model, timeout=timeout)
# results == { belief_id: {"verdict": ..., "reason": ..., "quote": ...} }
```

Inside `verify_beliefs`:

1. For each `(belief, source_content)` pair, render a markdown block.
2. Join blocks with `---` separators and substitute into `VERIFY_PROMPT`.
3. Invoke the model once (single batch call); no retry loop here.
4. `parse_verify_response` extracts and validates the JSON; returns the verified subset.

Because `verify_beliefs` does not itself call `read_source`, callers that *want* to skip reading a source (e.g., a source URL–only belief) can just pass `None` and the prompt will correctly steer the model to `INCONCLUSIVE`.

---

## Invariants

- **Source content is truncated** at exactly `MAX_SOURCE_CHARS` characters plus the truncation marker — downstream prompt size is bounded.
- **Verdicts are returned uppercased** but *not* validated against `{CONFIRMED, STALE, PARTIAL, INCONCLUSIVE}`. A model emitting `"MAYBE"` will be passed through as `"MAYBE"`. Callers relying on the four-value taxonomy must re-check.
- **Missing beliefs are silent**: if the model omits an ID, `parse_verify_response` simply doesn't include it. There is no per-ID placeholder for unverified beliefs.
- **Parser requires `verdict` be a string**; numeric verdicts or missing keys are dropped.
- **Empty response and non-JSON responses both yield `{}`** — indistinguishable from a total model refusal.
- **`read_source` never raises** and never returns a partial-then-raised failure; either content or `None`.
- **Greedy regex caveat**: `re.search(r"\{.*\}", ..., DOTALL)` matches from the *first* `{` to the *last* `}`. This is fine when the model emits exactly one JSON object, but will fail if the model emits prose containing additional braces after the JSON (e.g., another code block), because the resulting concatenation won't be valid JSON. The prompt's "Return ONLY a JSON object" instruction mitigates this but doesn't enforce it.

---

## Error Handling

- **I/O errors are swallowed**: `read_source` catches `OSError` and `UnicodeDecodeError` and returns `None`. A decoded-with-replacement strategy (`errors="replace"`) is intentionally paired with the try/except so legitimate encoding oddities usually survive rather than abort.
- **Parse errors are swallowed**: no exception escapes `parse_verify_response`; malformed model output degrades to an empty result set.
- **LLM errors are propagated**: `verify_beliefs` does *not* wrap `invoke_model`. If the model times out or the provider returns an error, the caller sees that exception directly. The `timeout=120` parameter is the only resilience knob at this layer.
- **Nothing validates the model's quote against the source text.** The prompt asks the model to be literal, but there is no post-hoc substring check — a caller that needs that guarantee must re-implement it.

---

## Topics to Explore

- [function] `reasons/check_stale.py:resolve_source_path` — How `verify.py` resolves a source string (and its `db_dir` base) into a filesystem `Path`; worth reading to understand URL vs. file precedence and relative paths.
- [function] `reasons/llm.py:invoke_model` — The LLM provider abstraction; explains what `model="claude"` selects and whether retries/streaming are on the table.
- [file] `reasons/api.py` — Where the `verify_*` helpers get composed into a batch operation; shows how `read_source` results and `verify_beliefs` output flow back into the API surface.
- [file] `tests/test_verify.py` — Documents the expected JSON shape, truncation behavior, and how `invoke_model` is mocked; a good place to see the intended contract in code.
- [general] `verdict-taxonomy-consumers` — Grep for `"CONFIRMED"`/`"STALE"`/`"PARTIAL"`/`"INCONCLUSIVE"` to see which downstream modules assume the four-value taxonomy and whether they defend against unknown values.

## Beliefs

- `verify-beliefs-returns-empty-dict-on-parse-failure` — `verify_beliefs` (via `parse_verify_response`) returns `{}` rather than raising when the model output is empty or not valid JSON.
- `read-source-requires-resolve-source-path` — `read_source` depends on `check_stale.resolve_source_path` for path resolution and has no independent path-resolution logic.
- `source-truncation-caps-at-30k-chars` — `read_source` truncates content at `MAX_SOURCE_CHARS` (30,000) and appends a `[... truncated ...]` marker.
- `verify-prompt-forces-verbatim-quote` — `VERIFY_PROMPT` explicitly instructs the model to copy the quote verbatim and to mark `INCONCLUSIVE` when source is missing, but the code does not verify the quote against the source.
- `parse-verify-response-normalizes-verdict-case` — `parse_verify_response` uppercases the `verdict` string but does not validate it against the four allowed values.
- `verify-beliefs-leaves-read-source-to-caller` — `verify_beliefs` accepts pre-read source content; it never calls `read_source` itself, so callers control disk I/O.
