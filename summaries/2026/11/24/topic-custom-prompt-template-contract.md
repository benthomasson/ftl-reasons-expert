---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-24
---

# Topic: Explores supported placeholders and brace-escaping behavior for `prompt_template`.

# Supported placeholders and brace-escaping for `prompt_template`

Based on the observed code, `prompt_template` is a custom prompt string for the derive flow. It is rendered with Python’s `str.format(...)`, not a custom templating language.

## Where `prompt_template` comes from

- `reasons/cli.py:1669` initializes `prompt_template = None`.
- `reasons/cli.py:1676` reads a user-provided template file with `prompt_path.read_text()`.
- The value is passed into the derive layer at `reasons/cli.py:1741` and `reasons/cli.py:1757`.
- `reasons/derive.py:443` accepts `prompt_template=None` as a parameter.
- `reasons/derive.py:539` selects the template:

```python
template = prompt_template or DERIVE_PROMPT
```

So `None` or an empty string falls back to the built-in `DERIVE_PROMPT`.

## Supported placeholders

Rendering happens at `reasons/derive.py:541`:

```python
prompt = template.format(...)
```

If the template references a key that was not supplied to `.format(...)`, Python raises `KeyError`. The code catches that and raises a `ValueError` with the message at `reasons/derive.py:553`:

```text
Custom prompt template references unknown placeholder: ...
```

The same error message enumerates the available placeholders at `reasons/derive.py:554–556`:

- `{beliefs_section}`
- `{derived_section}`
- `{total_in}`
- `{total_derived}`
- `{max_depth}`
- `{domain_context}`
- `{cross_agent_task}`
- `{agents_stats}`

So a custom template should use those names, e.g. `{beliefs_section}`. An unrecognized placeholder such as `{foo}` causes the `ValueError`. This is covered by `tests/test_derive.py:1088–1091`, which expects `ValueError` matching `"unknown placeholder"`.

## Brace escaping

Because rendering uses Python `.format(...)`, literal braces must be doubled:

- `{{` in the template becomes `{` in the rendered prompt.
- `}}` in the template becomes `}` in the rendered prompt.

`reasons/derive.py:562` explicitly instructs users:

```text
Use {{ and }} to include literal braces in the template.
```

That line is itself inside an f-string, so the source uses quadruple braces to display the double-brace syntax to the user.

This is important for custom templates containing JSON examples, JSON schemas, or other literal brace-delimited content. The same convention appears elsewhere in prompt constants that are later passed through `.format(...)`, for example:

- `reasons/repair.py:63`, `65`, `106`, `125` use `{{ ... }}` in prompt text; those prompts are formatted at `reasons/repair.py:173`, `195`, `250`, `262`.
- `reasons/propose_update.py:66`, `74` use doubled braces; the prompt is formatted at `reasons/propose_update.py:320`.
- `reasons/verify.py:32–38` uses doubled braces; the prompt is formatted at `reasons/verify.py:135`.
- Similar patterns appear in `reasons/review.py`, `reasons/review_justifications.py`, `reasons/review_premises.py`, and `reasons/repair_premises.py`.

## Test coverage

`tests/test_derive.py` includes relevant cases:

- `tests/test_derive.py:1076` builds a prompt with a custom `prompt_template`.
- `tests/test_derive.py:1084` builds with `prompt_template=None`, exercising the default.
- `tests/test_derive.py:1088–1091` verifies that an unknown placeholder raises `ValueError` matching `"unknown placeholder"`.
- `tests/test_derive.py:1092` and `1099` contain additional custom-template cases, though the observed snippet does not show their full assertions.

## Limitation of the observations

The snippet does not include the full `build_prompt` function body around `reasons/derive.py:539–562`, so the exact values assigned to each placeholder are not visible. However, the valid placeholder names are clearly enumerated in the error message at `reasons/derive.py:554–556`.

## Topics to Explore

- [function] `reasons/derive.py:build_prompt` — Inspect the exact keyword arguments passed to `template.format(...)` and how each supported placeholder value is constructed.
- [file] `tests/test_derive.py` — Read the full tests around lines 1076–1099 to confirm expected brace-escaping behavior and custom-template edge cases.
- [file] `reasons/derive.py` — Review `DERIVE_PROMPT` and the surrounding custom-template handling at lines 539–562.
- [file] `reasons/cli.py` — Trace the CLI option that reads `prompt_path` and passes it into the derive command at lines 1669–1757.
- [general] `prompt-template-escaping-pattern` — Compare all prompt constants that call `.format(...)` to see the project-wide convention for escaped JSON braces.

## Beliefs

- `prompt-template-uses-python-str-format` — Custom `prompt_template` rendering is performed by `template.format(...)` in `reasons/derive.py:541`, so placeholders follow Python `str.format` syntax.
- `prompt-template-supports-eight-named-placeholders` — The error message at `reasons/derive.py:554–556` enumerates `beliefs_section`, `derived_section`, `total_in`, `total_derived`, `max_depth`, `domain_context`, `cross_agent_task`, and `agents_stats`.
- `unknown-placeholder-raises-valueerror` — A missing placeholder triggers `ValueError` with “unknown placeholder” via `reasons/derive.py:553`, as tested at `tests/test_derive.py:1091`.
- `literal-braces-require-doubling` — Templates formatted with `.format(...)` must use `{{` and `}}` for literal braces, as instructed at `reasons/derive.py:562`.
- `cli-loads-prompt-template-from-file` — `reasons/cli.py:1676` reads the prompt template file text and passes it as `prompt_template` at `reasons/cli.py:1741` and `1757`.
