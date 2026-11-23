---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-23
---

# Topic: Look for project policy on backward compatibility, deprecation windows, and major-version removals.

# Backward Compatibility, Deprecation, and Major-Version Policy

## Bottom line

**There is no written policy in this repository.** The three targeted searches (`grep_backward_compat`, `grep_deprecation_code`, `grep_version_policy`) all returned `total_count: 0`, and none of the four documents surfaced (`CHANGELOG.md`, `README.md`, `CLAUDE.md`, `pyproject.toml`) contains a "Compatibility", "Deprecation", "Versioning", or "Contributing" section.

What *does* exist are **two implicit precedents** — a compatibility shim and a rename — plus a changelog format that has the vocabulary to record deprecations but has never used it.

---

## 1. The changelog format allows deprecation tracking, but nothing is tracked

`CHANGELOG.md` (140 lines) declares in its header that it follows [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/) and Semantic Versioning. That format defines six section headings — `Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security`.

Scanning the entire file, only **`Added`**, **`Changed`**, and **`Fixed`** appear:

| Heading | Appears in versions |
|---|---|
| `Added` | 0.18.0, 0.17.0, 0.15.0, 0.12.0, 0.7.0, 0.6.0, 0.5.0, 0.4.0, 0.2.0, 0.1.0 |
| `Changed` | 0.3.0 only (`Renamed from rms to reasons`) |
| `Fixed` | 0.16.0, 0.15.0, 0.14.0, 0.13.0, 0.6.0, 0.2.0 |
| `Deprecated` | **never** |
| `Removed` | **never** |

The absence of a single `Deprecated` or `Removed` entry across 140 lines of history is the strongest signal: **no feature has ever been formally deprecated, and no removal has ever been announced ahead of time.** The 0.3.0 rename is filed under `Changed` with no transition period described.

---

## 2. Backward compatibility is handled by shim, not by policy

The only real compatibility commitment in the codebase is the import-path alias, described in `CLAUDE.md`:

- **`CLAUDE.md`, "Project Structure" section**: `reasons_lib/` is annotated as a `backwards-compat shim — re-exports from reasons/`, and `reasons/` is described as "also aliased as `reasons_lib` for backwards compat".
- **`pyproject.toml:29–30`** (`[tool.setuptools]`): `packages = ["reasons", "reasons_lib"]` — the shim is a *first-class shipped package*, published in every wheel. That is a distribution-level guarantee that `import reasons_lib` keeps working.
- **`CHANGELOG.md`, 0.3.0**: `Renamed from rms to reasons (CLI, library, package)` + `Published to PyPI as ftl-reasons`.
- **`CLAUDE.md`, "What This Is"**: documents the rename rationale (tool-name ablation showed `rms` confused LLMs by 5 pp).

### The compat surface is asymmetric — and this is undocumented

| Surface | Old name | Shim provided? | Evidence |
|---|---|---|---|
| Python import | `reasons_lib` | ✅ Yes | `pyproject.toml:30`, `CLAUDE.md` structure block |
| CLI binary | `rms` | ❌ No | `pyproject.toml:26–27` `[project.scripts]` contains only `reasons` and `reasons-mcp` |
| PyPI distribution | `rms` | ❌ No | 0.3.0 entry says published as `ftl-reasons` |

So the project *did* keep the old Python import alive, but **silently dropped the `rms` console script**. Anyone with `rms retract foo` in a Makefile or CI script broke at 0.3.0 with no documented warning. There is no record in `CHANGELOG.md` of how long `reasons_lib` is intended to survive — no "will be removed in X" note, and no `DeprecationWarning` emitted from the shim (confirmed by `grep_deprecation_code`, 0 matches).

---

## 3. No major version exists yet, so "major-version removal" has no defined meaning

- `pyproject.toml:8`: `version = "0.58.0"`.
- Every changelog entry is `0.x.0`; the highest is `0.18.0`.

Under strict SemVer, `0.x` means the public API is **not yet stable** and breaking changes are permitted in minor bumps — which is exactly what 0.3.0 did (a rename in a `0.MINOR` release). Since 1.0.0 has never shipped, the question "what happens at a major-version removal?" is currently unanswerable from these observations: no 1.0 boundary has been crossed, so no precedent exists for removal across one.

There is also a **bookkeeping inconsistency worth flagging**: `pyproject.toml` claims `0.58.0` while the changelog terminates at `0.18.0` (40 minor versions of history are missing from the file that is supposed to record them). Any policy you infer from the changelog is therefore built on an incomplete record — removals in `0.19.0`–`0.58.0`, if any occurred, are invisible here.

---

## 4. A caveat on the negative search results

The "no matches" output is **partly a false negative**. The `grep_backward_compat` pattern is:

```
(?i)backward[- ]?compat|deprecat|breaking change|major (version|release|bump)|semver
```

The alternation `backward[- ]?compat` requires `backward` *immediately* followed by an optional hyphen/space and `compat`. The actual prose in `CLAUDE.md` is **`backwards-compat`** and **`backwards compat`** — with an `s` before the separator. The pattern does not match `backwards`, so it missed both occurrences.

What the searches legitimately establish:
- ✅ The word `deprecat` / `DeprecationWarning` appears nowhere (0 hits in `grep_deprecation_code`).
- ✅ The phrase `breaking change` and `semver`-as-policy appear nowhere.
- ⚠️ The claim "no backward-compatibility discussion exists" is **wrong** — `CLAUDE.md` discusses it twice; the regex just didn't match the plural form.

---

## 5. What's missing to answer the question properly

If you need an actual policy, these artifacts are not in the observation set:

1. **`CONTRIBUTING.md` / `docs/` / `GOVERNANCE.md`** — the usual home for a deprecation window ("N minor releases" or "never within a minor"). Not observed at all.
2. **Changelog entries 0.19.0 – 0.58.0** — the gap between `pyproject.toml:8` and the newest `CHANGELOG.md` entry.
3. **The `reasons_lib/` shim source itself** — to check whether it emits a warning and whether it lists *symbols* (re-export list) or uses `from reasons import *`. The latter leaks new names into the old namespace and is itself a compat decision.
4. **Git history / release tags** — to confirm whether 0.3.0 was a hard cut or shipped both binaries for a window (the `[project.scripts]` table shows only the *current* state, not past state).

**Verdict to give your colleague:** treat this project as "0.x, API-unstable, best-effort compatibility via shims, no deprecation contract." If you depend on it, pin exact versions and watch `CHANGELOG.md` — but note the changelog is currently 40 releases behind the package version, so pinning is the only safe contract.

---

## Topics to Explore

- [file] `reasons_lib/__init__.py` — Determines whether the shim is a curated re-export list (explicit compat contract) or a wildcard (accidental surface); check for `DeprecationWarning`.
- [file] `CHANGELOG.md` — Compare the last entry (`0.18.0`, 2026-04-25) against `pyproject.toml`'s `0.58.0` to size the undocumented release gap.
- [file] `pyproject.toml` — `[project.scripts]` and `[project.optional-dependencies]` define the real public surface: entry points, extras (`pg`, `mcp`, `llm`), and the `requires-python = ">=3.10"` floor are all compat commitments.
- [general] `release-tagging-convention` — Whether git tags exist per changelog version and whether a tag was cut for the `rms` → `reasons` rename; the only way to recover a historical compat window.
- [general] `deprecation-warning-strategy` — No `DeprecationWarning` exists anywhere today; a strategy would need to be introduced from scratch alongside a `CONTRIBUTING.md` policy.

---

## Beliefs

- `changelog-defines-no-deprecation-policy` — `CHANGELOG.md` follows Keep a Changelog but contains zero `Deprecated` and zero `Removed` sections across all 140 lines.
- `reasons-lib-is-a-backward-compat-shim` — `reasons_lib` is shipped as a first-class package in `pyproject.toml` (`packages = ["reasons", "reasons_lib"]`) to preserve the pre-0.3.0 import path.
- `rms-rename-has-no-cli-shim` — The 0.3.0 rename dropped the `rms` console script without a compatibility entry point; `[project.scripts]` lists only `reasons` and `reasons-mcp`.
- `pyproject-version-exceeds-changelog-latest` — `pyproject.toml` declares version `0.58.0` while the newest `CHANGELOG.md` entry is `0.18.0`, leaving 40 minor releases undocumented.
- `codebase-has-no-deprecation-warnings` — No `DeprecationWarning`, `deprecat*`, or `breaking change` string appears anywhere in the searched code or docs.
- `backward-compat-grep-pattern-misses-plural` — The `backward[- ]?compat` regex fails to match the actual `backwards-compat` wording in `CLAUDE.md`, producing a false-negative search result.
