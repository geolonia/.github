# Suggested commands

This is a docs/templates repo - no test runner, no build pipeline. Most
operations are `git` + `gh` + a markdown linter.

## Core workflow

```sh
# Branch from main
git checkout -b chore/<topic>          # or issue-<n>-<slug> per agent-policy.md

# Push and open PR
git push -u origin <branch>
gh pr create --base main --title "..." --body "..."

# Watch CodeRabbit + checks
gh pr checks <num> --watch
gh pr view <num> --json reviews,comments

# Cleanup after merge
git checkout main && git pull --ff-only origin main
git branch -d <branch>
git remote prune origin
```

## Lint markdown

`.markdownlint.json` configures the linter. Run it before committing
docs changes:

```sh
npx markdownlint-cli2 "docs/**/*.md" "*.md" "profile/*.md"
# or
npx markdownlint "docs/**/*.md" "*.md" "profile/*.md"
```

(There is no committed `package.json` - markdownlint runs ad-hoc via
`npx` or whatever the team has installed.)

## Validate workflow YAML

```sh
# Syntax check via actionlint (CodeRabbit also runs this on PR)
actionlint .github/workflows/*.yml workflow-templates/*.yml
```

## Tag management for reusable workflows

```sh
# Move v1 to current main
git tag -f v1
git push -f origin v1

# Inspect existing tags
git tag --list
gh api repos/geolonia/.github/git/refs/tags --jq '.[].ref'

# Cut a new major after a breaking change
git tag v2
git push origin v2
```

## TechDocs preview locally

```sh
pip install mkdocs-techdocs-core
mkdocs serve   # local preview at http://127.0.0.1:8000
```

## Inspect CodeRabbit central config rollout

```sh
# Confirm the canonical file is reachable
curl -fsSL https://raw.githubusercontent.com/geolonia/.github/main/.coderabbit.yaml | head

# List which org repos still have a local .coderabbit.yaml
gh search code 'org:geolonia path:.coderabbit.yaml -filename:.coderabbit.yaml'  # rough audit
```

## GitHub utilities

```sh
gh issue create --title "..." --body "..."
gh issue list --state open
gh issue close <num> --reason completed --comment "..."
gh pr list --label dependencies         # find Dependabot PRs
gh release list --limit 10
```

## macOS (Darwin) shell utilities

System is Darwin - BSD versions of common tools. Notable differences:

- `sed -i` requires an empty backup arg: `sed -i '' 's/a/b/' file`
- `find -E` for extended regex (BSD)
- `date -d` unavailable; use `gdate` (`brew install coreutils`) or `date -j`
- `pbcopy` / `pbpaste` for clipboard
