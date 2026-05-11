# Task completion checklist

Run before opening a PR. This repo has no automated test pipeline, so
the checklist relies on a few targeted manual checks.

## 1. Lint markdown (if you touched any .md)

```sh
npx markdownlint-cli2 "docs/**/*.md" "*.md" "profile/*.md"
```

Fix anything flagged before committing.

## 2. Validate workflow YAML (if you touched .github/workflows or workflow-templates)

```sh
actionlint .github/workflows/*.yml workflow-templates/*.yml
```

CodeRabbit will also run actionlint on the PR; fixing locally first
saves a review round.

## 3. Em-dash check

This repo's `AGENTS.md` forbids em-dashes (`-`) anywhere - markdown,
YAML, commit messages, PR text. Verify:

```sh
git diff --cached | grep $'\xe2\x80\x94' && echo "FAIL: U+2014 present" || echo "OK"
```

If you composed the PR body or issue body in a tool that auto-substitutes
hyphens to em-dashes (Apple Notes, Microsoft Word, etc.), check those
texts manually too.

## 4. Public-only content sweep

This repo is public. Scan the diff for anything that should not be:

```sh
git diff --cached | grep -iE 'secret|password|token|api[-_]?key|@<your-org-domain>'
```

Hits don't always indicate a leak (the word "secret" appears in
documentation), but every match deserves a deliberate look.

## 5. Tag-pinning sanity (if you touched workflow-templates)

If you changed a template that references a reusable workflow, confirm
the tag it points at exists:

```sh
gh api repos/geolonia/.github/git/refs/tags --jq '.[].ref'
```

## 6. CodeRabbit central-config sanity (if you touched .coderabbit.yaml)

This file is fetched by every other Geolonia repo. After merging, the
next PR opened in any repo will see the new settings. Before merging,
double-check:

- YAML parses (CodeRabbit will tell you on the PR; verify locally with
  `yamllint .coderabbit.yaml` if installed).
- No setting is too repo-specific - central config should apply org-wide.
- Plan a rollback: a quick revert PR is the right response if the new
  config produces noisy or wrong reviews.

## 7. Cross-impact of community-health edits

If you touched a community-health file (`CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`,
`SUPPORT.md`, `SECURITY.md`, `.github/ISSUE_TEMPLATE/*`, or
`.github/pull_request_template.md`), remember the change cascades to
every Geolonia repo that doesn't override locally. Higher-stakes than
a typical doc edit.

## 8. Open the PR

- Title under 70 chars, conventional-commit prefix.
- Body follows the org PR template - Summary, link to issue
  (`Fixes #N` / `Refs #N`), test plan.
- Wait for CodeRabbit review.
- Resolve every CodeRabbit thread after fixing (per global memory
  `global/issue-first-pr-workflow`).
- One human approval before merge.

## Red flags

- Any sensitive-looking string in the diff - abort and clean up first.
- Em-dashes anywhere in your changes.
- Bumping a tag (`v1`, `v2`) on a reusable workflow without a release
  note explaining what changed.
- Editing `.coderabbit.yaml` without a clear rollback plan.
