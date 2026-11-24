---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: How partial LLM failures degrade the wiki into raw-node rendering, and how callers should detect it.

# How partial LLM failures degrade the wiki into raw-node rendering

## The mechanism, step by step

`build_wiki` in `reasons/build_wiki.py:246` has two rendering modes, and it chooses between them **per group** (per topic page), not globally.

1. **Generation phase.** If `model` is non-empty, `build_wiki` calls `generate_wiki_page` for each group (`build_wiki.py:282`, via `reasons/build_wiki.py:generate_wiki_page`, which delegates to `reason/llm.py:invoke_model`). Successful content is stored in `generated_content[label]`. Failures are **caught locally and swallowed**:

   - Parallel branch (`build_wiki.py:~305-325`): the `try/except` around `future.result()` prints `WARN: {label} failed: {e} ({done}/{total_groups})` to stderr and does **nothing else**.
   - Sequential branch (`build_wiki.py:~326-335`): same pattern — `WARN: {label} failed: {e}` to stderr, nothing else.

   In both paths, a failed label is simply **never inserted** into `generated_content`. There is no sentinel value, no error list accumulated, no exception propagation.

2. **Rendering phase** (`build_wiki.py:~338-350`). For every group it does:

   ```python
   if label in generated_content:
       page_lines.append(generated_content[label])        # LLM prose
   else:
       for nid in sorted(nids):
           page_lines.append(_format_node(...))           # raw-node fallback
   ```

   So the **absence** of a key in `generated_content` is the trigger for the raw-node fallback path. That absence happens in exactly two situations:
   - `model == ""` (LLM intentionally not requested — see docstring at `build_wiki.py:253-254`), or
   - the LLM call for that page **raised any exception**: `FileNotFoundError`, `ImportError`, `RuntimeError`, `subprocess.TimeoutExpired`, `ValueError` (unknown model), etc. — all the exception types enumerated in `reasons/llm.py:invoke_model`'s docstring are caught by the bare `except Exception` in the render loop's generation branches.

3. **Result.** A partial LLM failure — e.g. one page times out while the rest succeed — produces a wiki where **some pages are synthesized prose and some pages are raw-node markdown**, side-by-side, with only stderr warnings as evidence. There is no gap/hole: each failed topic is replaced wholesale by the raw-node rendering of the same beliefs, so the wiki is structurally complete but semantically inconsistent.

## What "raw-node rendering" actually looks like

`_format_node` (`build_wiki.py:_format_node`) emits a very distinctive shape per node:

- `### {node_id}`
- `**Status:** IN|OUT`
- optionally `**Duplicate of:**`, `**Superseded by:**`, `**Defeated by:** {link} ({label})`
- the node's `text`
- `**Depends on:**` / `**Depends on (active):**` / `**Depends on (other):**`
- `**Supports:**`

Contrast with `WIKI_PAGE_PROMPT` (`build_wiki.py:WIKI_PAGE_PROMPT`), which asks the model for *prose*, `##`-level section headers, a leading overview paragraph, and belief IDs in parentheses — **no per-node `###` blocks and no `**Status:**` lines**.

So the degradation signature is unmistakable at the file level:

> A page whose body is composed almost entirely of `### <node-id>` headings followed by `**Status:**` and `**Depends on:**` lines is a fallback page, not an LLM page.

A subtle caveat: `_linkify` runs *after* the branch (`build_wiki.py:340-343`), so fallback pages still contain correct cross-page links. They look "finished" on superficial inspection, which is precisely why callers can't eyeball one link and conclude the page succeeded.

## Why callers can't detect it from the return value

`build_wiki` returns (`build_wiki.py:~315`):

```python
return {"output_dir": output_dir, "pages": len(groups), "total_nodes": total}
```

That dict has **no** `failed_groups`, `degraded_pages`, `llm_calls`, or `fallback_pages` key. `pages` is `len(groups)` regardless of whether every page was LLM-generated, none was, or half failed. `reasons/api.py:build_wiki` (`api.py:3562`, delegating at `api.py:3624`) and `reasons/cli.py:cmd_build_wiki` (`cli.py:2320`, calling `api.build_wiki` at `cli.py:2326`) therefore have **no programmatic signal** that degradation occurred. Detection must be done by inspection.

## How a caller should detect it — three practical checks

1. **Parse each generated page for the fallback signature.** After `build_wiki` returns, read each `*.md` (skipping `index.md`) and test for the `_format_node` markers:
   - a `### <node-id>` heading whose text matches a known node id,
   - immediately followed by `**Status:** IN` / `**Status:** OUT`,
   - and/or any `**Depends on:**`, `**Supports:**`, `**Defeated by:**` line.

   If such markers are present on the page **and** the page contains no `##` prose sections, the page was rendered by `_format_node` → it failed over. Do this per-file so you can identify *which* topics degraded.

2. **Capture stderr.** Both generation branches print `WARN: {label} failed: {e} ...` (`build_wiki.py:~312`, `~334`). Cli callers via `cli.py:cmd_build_wiki` only get the summary line `Wiki written to ...` on stdout, so stderr must be redirected/inspected. This is the *only* place the underlying exception message survives — the return dict discards it entirely.

3. **Cross-check against intent.** If you passed `model=None` / `model=""`, raw-node rendering is *expected* (docstring `build_wiki.py:246-254`, and `if model and parallel > 0` / `elif model` guards at `~300`, `~326`). So a page with raw-node markers is only a *failure* signal when `model` was non-empty. Make this distinction explicitly — otherwise "model not requested" and "all LLM calls failed" are indistinguishable from output alone.

An additional gap worth flagging: `_parse_cli_json` in `reasons/llm.py:_parse_cli_json` falls back to returning the **raw CLI stdout** when JSON parsing fails. If a CLI model returns malformed JSON, `build_wiki` will happily store and render that raw stdout (possibly containing cost metadata or error text) as the page's "LLM content" — a partially-broken page that does *not* match the raw-node signature, so check #1 alone will miss it. Detect this by asserting the generated content contains at least one of the expected prose markers (a leading paragraph, `##` sections) for pages you believe were LLM-generated.

## Why the tests don't catch it either

The visible portion of `tests/test_build_wiki.py` covers `_format_node` output extensively (`TestFormatNode`, lines ~119-257) and the happy path for `build_wiki` with `model=""` (`TestBuildWiki`, lines ~265-320). Tests patching `generate_wiki_page` start at `tests/test_build_wiki.py:469`, but the assertions in that region are truncated here — the point is that the *degradation path* (mixed success/failure producing a partially-raw wiki) is the case you should verify explicitly, because the codebase has no term for it (the search for `raw[-_ ]?node` returned zero matches) and no error contract around it.

---

## Topics to Explore

- [file] `reasons/build_wiki.py` — Read lines ~300-350 to see the exact exception swallowing and the per-group `if label in generated_content` fallback branch.
- [function] `reasons/build_wiki.py:_format_node` — The precise signature of raw-node output that callers must match on for detection.
- [function] `reasons/llm.py:invoke_model` — Enumerates every exception type that `build_wiki` catches and converts into a silent fallback page.
- [file] `tests/test_build_wiki.py` — Inspect lines 460-526 to see what failure-path assertions already exist (and which ones are missing) around `generate_wiki_page` exceptions.
- [general] `wiki-output-contract` — There is no return-value field distinguishing generated vs. fallback pages; consider adding `failed_groups` to the `build_wiki` return dict as the cleanest fix.

## Beliefs

- `build-wiki-catches-all-llm-exceptions` — `build_wiki` wraps every `generate_wiki_page` call in `except Exception` and converts the failure into a stderr `WARN:` line rather than propagating or recording it.
- `generated-content-membership-controls-rendering-mode` — A topic is rendered as LLM prose if and only if its label is a key in `generated_content`; otherwise the whole page falls back to `_format_node` output.
- `build-wiki-return-dict-has-no-degradation-signal` — `build_wiki` returns `{"output_dir", "pages", "total_nodes"}` with no field indicating how many pages fell back to raw-node rendering.
- `model-empty-and-model-failed-produce-identical-output` — A wiki built with `model=""` and a wiki where every LLM call raised produce structurally identical raw-node pages.
- `raw-node-output-is-identifiable-by-status-marker` — `_format_node` emits `**Status:**` and per-node `###` headings that `WIKI_PAGE_PROMPT` explicitly does not request, giving callers a detectable fingerprint.
- `parse-cli-json-returns-raw-stdout-on-parse-error` — `_parse_cli_json` returns the unparsed CLI output when JSON decoding fails, which `build_wiki` will treat as successful LLM content.
