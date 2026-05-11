# Code style and conventions

Repo is markdown + YAML. The hard rules are short.

## Markdown

- Configured by `.markdownlint.json`:
  - `MD033: false` - inline HTML allowed (used in some templates)
  - `MD041: false` - first-line H1 not required (some files start with
    intro paragraphs)
- All other markdownlint defaults apply: ATX headings, fenced code
  blocks, sentence-case headings, etc.
- File encoding: UTF-8 (per `.editorconfig`).
- Line endings: LF.
- Trim trailing whitespace; final newline.

## **No em-dashes** (org-wide rule)

The repo's `AGENTS.md` explicitly states: **"Must not use long dash `-`"**.

Applies to:

- Markdown files (docs/, profile/, root .md files)
- YAML descriptions and comments
- Commit messages
- PR titles and bodies
- Issue bodies and comments

Use a hyphen `-` or rephrase the sentence instead. This rule appears
to be unique to this repo (other Geolonia repos don't enforce it), so
when working **here** specifically, run a final check before pushing:

```sh
git diff --cached | grep -E '\xe2\x80\x94' && echo "FAIL: em-dash present" || echo "OK"
```

## YAML

- 2-space indent.
- Lowercase boolean values (`true` / `false`).
- Quote string values that contain special characters or could be
  misparsed as booleans / numbers (e.g. version-like strings).

## Workflow templates (`workflow-templates/*.yml`)

- Keep `<name>.yml` and `<name>.properties.json` consistent - the
  `.properties.json` controls what shows up in GitHub's "New workflow"
  picker.
- Templates should reference reusable workflows by **tag**
  (`@v1`, `@v2`), not by `@main` and not by SHA. See the
  `reusable_workflows` memory.

## Commit and PR text

- Conventional-commit prefixes: `chore:`, `feat:`, `fix:`, `docs:`,
  `test:`. Match what's already in `git log`.
- PR title under 70 chars.
- PR body follows the org template (`.github/pull_request_template.md`)
  - Summary, link to issue (`Fixes #N` or `Refs #N`), test plan.

## Prose

- English for all docs except where Japanese parallel text already
  exists (e.g. `profile/README.md`). Keep both languages in sync when
  one changes.
- `languagetool` is enabled in CodeRabbit at default level - keeps
  prose tight without being picky.
