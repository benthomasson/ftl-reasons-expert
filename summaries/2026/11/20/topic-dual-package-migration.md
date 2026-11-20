---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Survey how many `reasons_lib/*.py` files are pure shims versus real implementations; that ratio determines whether `reasons_lib` can be deleted wholesale or needs a staged deprecation.

## Survey result

The observed `reasons_lib` package contains **5 Python files**, and all 5 are pure re-export shims. There is no local implementation code.

| File | Shim statement | Real `def`/`class` | Deprecation hook | Classification |
|---|---|---:|---:|---|
| `reasons_lib/__init__.py` | Returned-content line 2: `from reasons import *` | 0 | 0 | Pure shim |
| `reasons_lib/api.py` | Returned-content line 2: `from reasons.api import *` | 0 | 0 | Pure shim |
| `reasons_lib/storage.py` | Returned-content line 2: `from reasons.storage import *` | 0 | 0 | Pure shim |
| `reasons_lib/metadata.py` | Returned-content line 2: `from reasons.metadata import *` | 0 | 0 | Pure shim |
| `reasons_lib/cli.py` | Returned-content line 2: `from reasons.cli import *` | 0 | 0 | Pure shim |

The aggregate scans agree with the file contents on the important points:

- `real_definitions_in_reasons_lib.total_count = 0` — no `def` or `class` declarations were found in `reasons_lib`.
- `deprecation_markers_in_reasons_lib.total_count = 0` — there are no current `DeprecationWarning`, `deprecated`, `warnings.warn`, or `__getattr__` deprecation hooks.
- `reasons_lib/__init__.py` explicitly describes itself as a “Backwards-compatibility shim — use `import reasons` instead.”

One inconsistency should be called out: `shim_imports_in_reasons_lib.total_count` is `0`, but the raw file reads show five `from reasons... import *` statements. The file contents are direct evidence, so the ratio should be based on those: **5 pure shims : 0 real implementations** — i.e. **100% pure shims**.

## Deletion implication

Because `reasons_lib` has no real implementation, the codebase does **not** need a staged deprecation to migrate hidden logic out of it. Wholesale deletion of the `reasons_lib` directory is structurally safe from the package’s own contents.

However, the package exists for backwards compatibility, so the deletion decision depends on remaining importers:

- If no internal or external code imports `reasons_lib`, it can be deleted wholesale.
- If downstream or external code still imports `reasons_lib`, deleting it wholesale is a breaking change. In that case, use a staged deprecation first — currently none exists — then remove after importers migrate to `reasons` / `reasons.api` / etc.

So: **the shim-to-real ratio says “delete wholesale after an import-usage audit”; it does not require staged deprecation because of implementation code. It may require staged deprecation only because of compatibility obligations.**

## Topics to Explore

- [general] `reasons-lib-import-sites` — Search the repo and published packages for `import reasons_lib` and `from reasons_lib` to decide whether wholesale deletion breaks callers.
- [file] `reasons_lib/__init__.py` — Confirm the root shim re-exports the complete legacy API and does not rely on `__all__` differences.
- [repo] `reasons` — Compare the public API of `reasons` against the legacy `reasons_lib` surface.
- [general] `public-api-deprecation-policy` — Determine whether `reasons_lib` is an externally supported path that requires warnings before removal.
- [diff] `remove-reasons_lib` — Draft the deletion plus import rewrites to `reasons.*` for any remaining internal usages.

## Beliefs

- `reasons-lib-is-pure-shim-package` — All five observed `reasons_lib/*.py` files are pure re-export shims with no local functions or classes.
- `reasons-lib-init-reexports-reasons-api` — `reasons_lib/__init__.py` line 2 re-exports the top-level `reasons` API via `from reasons import *`.
- `reasons-lib-submodules-map-one-to-one` — `api.py`, `storage.py`, `metadata.py`, and `cli.py` each re-export from the same-named `reasons.*` module.
- `reasons-lib-has-no-deprecation-hooks` — The observed shims contain no `DeprecationWarning`, `warnings.warn`, or `__getattr__` deprecation mechanism.
- `reasons-lib-deletion-requires-import-audit` — Wholesale deletion is safe from `reasons_lib`’s contents but requires confirming no remaining `reasons_lib` importers.
