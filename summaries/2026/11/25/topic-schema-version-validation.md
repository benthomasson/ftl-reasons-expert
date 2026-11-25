---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-25
---

# Topic: Check whether any importer rejects a payload whose `schema_version != "1.0"`; `import_into_network` currently just copies it into `network.meta`

## Short Answer

**No.** None of the importers inspected in the observations reject a payload whose `schema_version != "1.0"`. Every call site that touches `schema_version` either **copies it verbatim** into metadata, **provides it as a default** when absent, or **asserts its value in tests**. There is no comparison against a live `SCHEMA_VERSION` constant (which is `"1.0"` per `reasons/metadata.py:49` and `tests/test_metadata.py:19`), and no `raise`/error path associated with a version mismatch anywhere in the grep results.

## Evidence

### 1. `import_into_network` copies frontmatter as-is

`reasons/import_beliefs.py:171` defines `import_into_network`. Its body:

```python
beliefs_text, frontmatter = strip_frontmatter(beliefs_text)

if frontmatter:
    for key in ("schema_version", "project_name", "created_at"):
        if key in frontmatter and frontmatter[key]:
            network.meta[key] = frontmatter[key]
```

The grep hit at `reasons/import_beliefs.py:183` locates this loop. The only gate is `if key in frontmatter and frontmatter[key]` — i.e., "present and non-empty." A payload with `schema_version: "0.9"` or `schema_version: "2.0"` will be stored as `network.meta["schema_version"]` exactly as supplied.

`strip_frontmatter` (in the same file) just parses lines of `key: value`, strips quotes, and returns a dict. It doesn't know or care about the schema version.

### 2. Other importers follow the same pattern

The `schema_version` tuple copy appears identically in:

- `reasons/api.py:2127` — the same `for key in ("schema_version", "project_name", "created_at")` loop, in the JSON import path (surrounding function likely `import_json` — see the test file `tests/test_import_json.py`). No adjacent version comparison appears in the grep, and a comparison would necessarily include a `schema_version` string match.
- `reasons/pg.py:2250` and `reasons/pg.py:2398` — the same tuple-loop appears twice in the Postgres backend, giving two more copy-without-validation paths.

### 3. Defaults only fill missing values

- `reasons/storage.py:326` — `meta.setdefault("schema_version", SCHEMA_VERSION)`. This only inserts `SCHEMA_VERSION` (`"1.0"`) when the key is **absent**. If the source payload says `"0.9"`, it's preserved.
- `reasons/metadata.py:49` — `"schema_version": SCHEMA_VERSION` builds the *canonical* meta dict for freshly-created networks. Not used to validate imported data.

### 4. Tests assume `"1.0"` but never assert rejection

- `tests/test_metadata.py:19` — `assert meta["schema_version"] == "1.0"`
- `tests/test_export_card.py:47` — same assertion against `SCHEMA_VERSION`
- `tests/test_import_hf.py:17` and `tests/test_hf_default_org.py:20` — fixtures hard-code `"schema_version": "1.0"` in input
- `tests/test_import_json.py:6` — the round-trip tests import JSON, but there is **no test that feeds a mismatched `schema_version` and expects an error**.

So the behavior currently encoded in tests is "copy it through."

## What's missing / caveats

- The **full body** of `api.import_json` in `reasons/api.py` is not shown — only line 2127. If a guard existed before the loop that raised on a version mismatch, it would still need to reference `"schema_version"` as a string and would appear in the grep. Since it doesn't, I'm confident no rejection exists, but a peek at the surrounding function would settle it definitively.
- The two `pg.py` loops (lines 2250, 2398) also lack surrounding function context. Their behavior matches the copy pattern.
- We have not seen the **definition** of `SCHEMA_VERSION` itself (in `reasons/metadata.py` around line 37–49, inferred to be `"1.0"` from tests). If you later add validation, that's the constant to compare against.

## Implication

If you want to enforce `schema_version == "1.0"`, you'd need to add the check in **four places** (`import_beliefs.py:183`, `api.py:2127`, `pg.py:2250`, `pg.py:2398`), or better, factor a helper like `validate_schema(frontmatter)` in `reasons/metadata.py` and call it from each import path. Keeping the current copy-through behavior is a deliberate (or at least consistent) design choice: the code treats `schema_version` as opaque metadata, not a gate.

## Topics to Explore

- [file] `reasons/api.py` — Confirm the full body of `import_json` around line 2127 and check whether a validation guard exists before the meta-copy loop
- [function] `reasons/api.py:import_json` — The primary JSON import path exercised by `tests/test_import_json.py`; verify no schema-version branch was hidden by context truncation
- [function] `reasons/import_beliefs.py:import_into_network` — The function in question; add/remove schema-version validation here to change the behavior
- [file] `reasons/metadata.py` — Home of the `SCHEMA_VERSION` constant and metadata defaults; the natural place to add a `validate_schema_version()` helper
- [general] `schema-version-migration-strategy` — If you intend to reject `!= "1.0"`, you need a plan for how version bumps (e.g. `"2.0"`) migrate existing on-disk registries

## Beliefs

- `import-into-network-copies-schema-version-verbatim` — `import_into_network` stores whatever `schema_version` appears in the source frontmatter directly into `network.meta["schema_version"]`
- `no-importer-rejects-schema-version-mismatch` — None of the four import sites (`import_beliefs.py:183`, `api.py:2127`, `pg.py:2250`, `pg.py:2398`) compare `schema_version` against `"1.0"` or `SCHEMA_VERSION`
- `schema-version-default-is-fill-only` — `storage.py:326` uses `meta.setdefault("schema_version", SCHEMA_VERSION)`, so a supplied non-default version survives
- `schema-version-constant-is-1-0` — `SCHEMA_VERSION` (defined in `reasons/metadata.py`) is `"1.0"`, as asserted by `tests/test_metadata.py:19` and `tests/test_export_card.py:47`
- `import-meta-copy-tuple` — The `("schema_version", "project_name", "created_at")` tuple is duplicated verbatim across `import_beliefs.py`, `api.py`, and `pg.py`, indicating a shared, copy-through metadata policy
