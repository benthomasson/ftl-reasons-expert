---
model: "ollama:qwen-64k:latest"
date: 2026-11-01
---

# File: reasons/metadata.py

# `reasons/metadata.py`

## 1. Purpose

This module is the **single source of truth for the metadata envelope** that wraps every exported or stored belief-network artifact. Whenever a project is exported to JSON, rendered as Markdown, or persisted into SQLite/PostgreSQL, the wrapping metadata dict must have identical field names, identical defaults, and a consistent generator string. Rather than letting each export/storage path define its own ad-hoc dict (which inevitably drifts), this module centralizes the construction in one place: `build_meta()`.

Think of it as the "stamped header" on every document the system produces. It owns:

- The schema version constant (`SCHEMA_VERSION`)
- The generator identification string (package name + resolved version)
- The exact set of keys, their types, and their default semantics

It has **no knowledge of what the payload is** — it only produces the envelope.

## 2. Key Components

### `SCHEMA_VERSION = "1.0"`
A module-level constant. Every metadata dict carries this value under the key `"schema_version"`. If you add fields or change semantics, this is the value you bump, and downstream consumers (importers, validators) can branch on it.

### `_get_generator() -> str`
A private helper (underscore-prefixed) that returns a string like `"ftl-reasons/0.40.0"`. It works by calling `importlib.metadata.version("ftl-reasons")`, which reads the installed package's version from the environment (PEP 345 / `importlib.metadata`). This avoids hardcoding the version anywhere.

Key detail: the import is **inside** the function body, not at module top level. This means the module can be imported and `build_meta` can be called even if the package metadata isn't available (e.g., during `python -c` scripts, frozen builds, or broken installs) — it degrades gracefully to `"ftl-reasons/unknown"`.

### `build_meta(project_name="", node_count=0, created_at="") -> dict`
The public API. Returns a flat `dict` with exactly six keys:

| Key | Type | Default / Semantics |
|---|---|---|
| `schema_version` | `str` | Always `SCHEMA_VERSION` |
| `project_name` | `str` | Caller-supplied; `""` if omitted |
| `created_at` | `str` | Caller-supplied ISO-8601, **or** current UTC time if empty — this is the "preserved if set" contract |
| `updated_at` | `str` | **Always** current UTC time (never caller-supplied) |
| `node_count` | `int` | Caller-supplied; `0` if omitted |
| `generator` | `str` | Output of `_get_generator()` |

The `created_at` parameter has a subtle contract: if the caller passes a non-empty string, it is treated as the **original creation timestamp** and preserved verbatim. If it's empty (the default), the current time is used. This means on re-export of an existing project, you pass the old `created_at` to keep it stable, while `updated_at` always advances.

The timestamp format is ISO-8601 with second precision (`timespec="seconds"`), UTC, e.g. `"2025-01-15T09:32:01+00:00"`.

## 3. Patterns

- **Single Source of Truth / Facade**: The module docstring explicitly states this intent. Six different consumer files (`api.py`, `cli.py`, `export_card.py`, `export_markdown.py`, `pg.py`, `storage.py`) all call `build_meta()` rather than assembling the dict inline. This eliminates a classic drift bug where one exporter adds a `revision` field and another doesn't.

- **Defensive version resolution**: `_get_generator()` uses a `try/except Exception` around `importlib.metadata.version()`. This is the standard pattern for "best-effort version discovery" — you want the real version in production but don't want a `MetadataNotFoundError` to crash an export in a development or frozen environment.

- **Lazy import**: `importlib.metadata` is imported inside the function, not at module scope. This keeps the module's top-level import surface to just `datetime` and `timezone`, which are always available in the stdlib. It also means the module can be imported in environments where the package distribution metadata is absent.

- **Flat dict return type**: No dataclass, no `NamedTuple`, no ORM model. The return value is a plain `dict`, which is directly JSON-serializable and SQL-mappable. This is a deliberate choice for a module whose output is immediately consumed by serialization layers.

- **Duplicate mirror**: Notice `reasons_lib/metadata.py` exists in the repo tree. The `reasons/` and `reasons_lib/` directories appear to be parallel packages (possibly a source layout vs. a vendored/installed copy). The import list confirms `reasons_lib/metadata.py` also imports `reasons/metadata.py`'s symbols, suggesting one may re-export from the other.

## 4. Dependencies

### Upstream (this file imports)
- `datetime.datetime`, `datetime.timezone` — stdlib, for UTC timestamp generation
- `importlib.metadata.version` — stdlib, for PEP 345 package version resolution (lazy, inside `_get_generator`)

### Downstream (imports this file)
| Consumer | Likely use |
|---|---|
| `reasons/api.py` | Attaches metadata to API response payloads |
| `reasons/cli.py` | Stamps CLI output files with metadata |
| `reasons/export_card.py` | Wraps card exports in the metadata envelope |
| `reasons/export_markdown.py` | Prepend/append metadata block in `.md` output |
| `reasons/pg.py` | Writes metadata into a Postgres table/column |
| `reasons/storage.py` | Persists metadata alongside belief data in SQLite |
| `tests/test_metadata.py` | Unit tests for `build_meta` contracts |
| `tests/test_metadata_extended.py` | Edge-case tests (empty strings, unicode, large counts) |
| `tests/test_export_card.py` | Integration test verifying the card includes correct metadata |

The fan-in is wide (6 production modules + 3 test files), which reinforces the "single source of truth" role. A field-name typo here would silently corrupt every export format simultaneously.

## 5. Flow

Execution is trivially linear — there is no branching, no loops, no state:

```
caller invokes build_meta(project_name, node_count, created_at)
  │
  ├──► datetime.now(timezone.utc).isoformat(timespec="seconds")
  │    → now  (e.g. "2025-01-15T09:32:01+00:00")
  │
  ├──► _get_generator()
  │    ├──► importlib.metadata.version("ftl-reasons")   # succeeds
  │    │    → v = "0.40.0"
  │    └──► except → v = "unknown"
  │    → "ftl-reasons/0.40.0"
  │
  └──► return {
         "schema_version": "1.0",
         "project_name":   project_name,
         "created_at":     created_at or now,   # ← the only conditional
         "updated_at":     now,
         "node_count":     node_count,
         "generator":      "ftl-reasons/0.40.0"
       }
```

The only branch in the entire file is `created_at or now`: if the caller passed a non-empty string, it wins; otherwise the freshly computed `now` is used. Both `created_at` and `updated_at` resolve to the same value on a first-time export, but diverge on re-exports.

## 6. Invariants

- **Key set is fixed**: The returned dict always has exactly these six keys: `schema_version`, `project_name`, `created_at`, `updated_at`, `node_count`, `generator`. No extra keys, no missing keys. Consumers can rely on this shape without defensive `.get()` calls.
- **`updated_at` ≥ `created_at`**: On any single call, if `created_at` is empty they're equal. If `created_at` is a past timestamp, `updated_at` (always `now`) is ≥ it. This invariant is structural, not enforced by a check.
- **`schema_version` is immutable within a call**: It's always the module-level constant. A caller cannot override it by passing a dict.
- **`generator` always contains the package name** `"ftl-reasons/"` followed by either a version string or `"unknown"`. The doctest `assert "ftl-reasons/" in meta["generator"]` encodes this.
- **Timestamps are always UTC, second-precision, ISO-8601**. No other format is produced.

## 7. Error Handling

The error surface is deliberately minimal:

- **`_get_generator()`**: Catches **all** exceptions from `importlib.metadata.version()` and substitutes `"unknown"`. This includes `importlib.metadata.PackageNotFoundError`, `ImportError` (if `importlib` itself is broken), and any other unexpected failure. The rationale: metadata stamping is cosmetic; it must never take down an export. The `except Exception` (not `except BaseException`) means a `KeyboardInterrupt` would still propagate, which is correct.
- **`build_meta()`**: No exceptions are raised for any input. Passing `node_count=-1`, `project_name=None`, or a non-ISO string for `created_at` will not trigger a validation error — the function is a **formatter, not a validator**. If `created_at` is `None`, the `or` expression catches it (`None or now` → `now`). If it's a non-empty non-ISO string, it's passed through verbatim, which is the caller's responsibility.
- **No `raise` statements exist in this file.** Errors are either swallowed (version lookup) or delegated to the caller (input validation).

---

## Topics to Explore

- [file] `reasons/export_markdown.py` — See how `build_meta()` output is rendered into a Markdown front-matter or preamble block; this is where the dict keys become human-readable labels.
- [file] `reasons/pg.py` — Understand how the same metadata dict is mapped onto Postgres columns (or a JSONB column), and whether the schema there adds constraints the Python side doesn't enforce.
- [file] `tests/test_metadata_extended.py` — Edge-case coverage: unicode project names, zero/negative `node_count`, very old `created_at` values, and the `created_at` preservation contract.
- [function] `reasons/metadata.py:_get_generator` — In a frozen/`pyinstaller` build or a `pip install -e .` dev environment, verify what `importlib.metadata.version("ftl-reasons")` actually resolves to; the `"unknown"` fallback path may be more common than expected.
- [repo] `reasons_lib/metadata.py` — Compare against the `reasons/` copy to understand whether `reasons_lib` is a re-export shim, a vendored duplicate, or the "real" package that `reasons/` wraps.

## Beliefs

- `build-meta-returns-exactly-six-keys` — `build_meta()` always returns a dict with exactly the keys `schema_version`, `project_name`, `created_at`, `updated_at`, `node_count`, `generator`; no more, no fewer.
- `created-at-preservation-enables-stable-identity` — Passing a non-empty `created_at` to `build_meta()` preserves the original creation timestamp across re-exports, while `updated_at` always advances to the current time, enabling consumers to distinguish "when was this first made" from "when was it last touched."
- `generator-fallback-prevents-export-crash` — If `importlib.metadata.version("ftl-reasons")` raises for any reason (missing dist-info, frozen binary, broken install), `_get_generator()` returns `"ftl-reasons/unknown"` instead of propagating the exception, ensuring no export path crashes due to version-resolution failure.
- `schema-version-maps-to-consumer-migration` — The `SCHEMA_VERSION` constant (currently `"1.0"`) is the single field that downstream importers/validators use to decide whether they can parse the payload; bumping it signals a breaking change in the metadata or belief schema.
- `metadata-module-has-no-domain-logic` — `metadata.py` imports nothing from the `reasons` or `reasons_lib` packages; it depends only on `datetime` and `importlib.metadata`, making it safe to import in any context (CLI, server, test) without triggering side effects or circular imports.
