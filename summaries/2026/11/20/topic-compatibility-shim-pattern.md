---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-20
---

# Topic: The broader migration story: how the repository plans to retire `reasons_lib/` and whether there is a deprecation timeline (check `CHANGELOG.md` and `CLAUDE.md`).

## Migration status of `reasons_lib/`

Based on the provided docs, the migration is best understood as **already completed at the import-name level, with `reasons_lib/` retained only as a compatibility layer** — but there is **no documented deprecation or removal timeline**.

Key evidence:

- `CLAUDE.md:24` says `reasons/` is importable as `import reasons`, and is **“also aliased as `reasons_lib` for backwards compat.”**
- `CLAUDE.md:35` describes `reasons_lib/` explicitly as a **“backwards-compat shim — re-exports from `reasons/`.”**
- `CHANGELOG.md:136` mentions `reasons_lib.api` only in the historical `0.1.0` entry: “Python API layer (`reasons_lib.api`) returning dicts...”
- `pyproject.toml` still packages both names:
  ```toml
  [tool.setuptools]
  packages = ["reasons", "reasons_lib"]
  ```

So the current story appears to be: `reasons` is canonical; `reasons_lib` exists so old imports continue to work. There is no evidence in `CHANGELOG.md` or `CLAUDE.md` that a formal retirement is scheduled.

## Is there a deprecation timeline?

**No deprecation timeline is documented in the observed files.**

The grep for `deprecat|retire|migration|timeline|reasons_lib` across the docs returned only the same three `reasons_lib` mentions:

- `CHANGELOG.md:136` — historical `reasons_lib.api` mention.
- `CLAUDE.md:24` — alias for backwards compatibility.
- `CLAUDE.md:35` — backwards-compat shim description.

None of those state a removal version, deprecation warning, migration deadline, or replacement schedule. The grep for `reasons_lib` in Python files also returned zero matches, but that alone does not prove the shim is unused internally; it just means the string `reasons_lib` does not appear in the scanned Python source.

One important caveat: `CHANGELOG.md`’s latest entry is `0.18.0`, while `pyproject.toml` reports version `0.58.0`. The changelog may be stale relative to the package version, so absence of a deprecation note in `CHANGELOG.md` is not fully conclusive for the most recent releases. To establish a definitive timeline, you would need to inspect the actual `reasons_lib/__init__.py` shim, release notes/issues, or project README.

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Check whether the shim only re-exports from `reasons/` or emits a `DeprecationWarning`.
- [file] `CHANGELOG.md` — Compare the `0.1.0` `reasons_lib.api` mention with the `0.3.0` rename entry to reconstruct the naming migration.
- [file] `pyproject.toml` — Confirm that both `reasons` and `reasons_lib` are still packaged and distributed.
- [general] `reasons-lib-deprecation-issues` — Search repository issues/PRs for any planned removal or migration discussions.
- [file] `README.md` — Check which import path the public documentation recommends for new consumers.

## Beliefs

- `reasons-lib-is-backwards-compat-shim` — `reasons_lib/` is documented as a backwards-compat shim that re-exports from `reasons/` (`CLAUDE.md:35`).
- `canonical-import-is-reasons` — The canonical import path is `reasons`; `reasons_lib` is retained as an alias for backwards compatibility (`CLAUDE.md:24`).
- `no-deprecation-timeline-documented` — Neither `CHANGELOG.md` nor `CLAUDE.md` states a deprecation or removal timeline for `reasons_lib/`.
- `pyproject-still-ships-reasons-lib` — `pyproject.toml` still includes both `reasons` and `reasons_lib` in `[tool.setuptools] packages`.
- `changelog-mentions-reasons-lib-only-historically` — `CHANGELOG.md:136` mentions `reasons_lib.api` only in the `0.1.0` entry; later changelog entries do not mention `reasons_lib`.
