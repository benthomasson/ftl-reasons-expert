---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: Confirm whether *every* file under `reasons_lib/` follows the same shim pattern and how the migration is tracked (CHANGELOG.md, deprecation policy).

## Answer

No — the observations do **not** support confirming that *every* file under `reasons_lib/` follows the same shim pattern.

### What is confirmed

Three files were read and all use the same one-line star re-export style:

- `reasons_lib/__init__.py` line 1: `from reasons import *  # noqa: F401,F403`
- `reasons_lib/api.py` line 1: `from reasons.api import *  # noqa: F401,F403`
- `reasons_lib/llm.py` line 1: `from reasons.llm import *  # noqa: F401,F403`

So for these three, the pattern is: re-export the corresponding `reasons` package/module with a star import and suppress lint warnings. None of these sampled files contains a `DeprecationWarning`, `warnings.warn`, or extra compatibility logic.

### Why “every file” is not confirmed

`reasons_lib/` contains **33 total entries** (`list_reasons_lib_files.total_entries: 33`), but only 3 files were read. The remaining 30 files — e.g. `build_wiki.py`, `mcp_server.py`, `storage.py`, `verify.py` — are not shown, so we cannot verify that they all follow the same shim pattern.

The grep evidence is also unreliable: `grep_shim_patterns` reports `total_count: 0` for a pattern that includes `from reasons`, yet the read files clearly contain `from reasons...`. That means the grep result is inconsistent with the sampled file contents and should not be treated as proof of absence.

### How migration/tracking is documented

- `CLAUDE.md:35` explicitly documents `reasons_lib/` as a “backwards-compat shim — re-exports from `reasons/`”.
- `CLAUDE.md:24` says the package is importable as `reasons` and “also aliased as reasons_lib for backwards compat”.
- `CHANGELOG.md:136` mentions `reasons_lib.api` only under the 0.1.0 “Added” section as “Python API layer (`reasons_lib.api`) returning dicts for CLI and tool-call consumers”.
- The `0.3.0` changelog entry documents a rename from `rms` to `reasons`, but it does not document `reasons_lib` as a shim or describe a deprecation timeline.
- The doc grep for `deprecat|migrat|reasons_lib|shim` found no deprecation-policy wording. The only “shim” match is the `CLAUDE.md:35` internal documentation.
- `grep_tests_for_shim` found no tests referencing `reasons_lib`, `DeprecationWarning`, `deprecated`, or `shim`.

### Conclusion

The sampled files follow a consistent star-import shim pattern, and `CLAUDE.md` documents `reasons_lib/` as a backwards-compatibility alias. However, there is **not enough evidence** to say every file in `reasons_lib/` follows that pattern. Migration is only informally tracked in `CLAUDE.md`; `CHANGELOG.md` does not appear to track the shim migration, and no deprecation policy or runtime deprecation warnings were observed.

## Topics to Explore

- [general] `reliable-reasons-lib-shim-scan` — Run a corrected/complete grep over all 33 `reasons_lib/` files for `^from reasons(\.| import)` to verify the shim pattern reliably.
- [file] `reasons_lib/build_wiki.py` — One of the unread files; check whether it follows the same `from reasons.build_wiki import *` pattern.
- [file] `reasons_lib/mcp_server.py` — Another unread file; verify it has no extra compatibility logic or side effects.
- [file] `CLAUDE.md` — Lines 24 and 35 contain the only observed documentation of `reasons_lib` as a backwards-compat shim.
- [general] `deprecation-policy` — Search for any policy defining when `reasons_lib` will be removed and whether runtime warnings are expected.

## Beliefs

- `reasons-lib-is-a-backwards-compat-shim` — `CLAUDE.md:35` documents `reasons_lib/` as a backwards-compat shim that re-exports from `reasons/`.
- `reasons-lib-init-reexports-reasons` — `reasons_lib/__init__.py` line 1 is `from reasons import *  # noqa: F401,F403`.
- `reasons-lib-api-reexports-reasons-api` — `reasons_lib/api.py` line 1 is `from reasons.api import *  # noqa: F401,F403`.
- `reasons-lib-llm-reexports-reasons-llm` — `reasons_lib/llm.py` line 1 is `from reasons.llm import *  # noqa: F401,F403`.
- `changelog-lacks-reasons-lib-shim-deprecation-entry` — `CHANGELOG.md:136` mentions `reasons_lib.api` only as an added API layer; no changelog entry documents the shim or a deprecation policy.
