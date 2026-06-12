# Route issue to team board (`route-issue.yml`)

Opt a repo in to **issue auto-routing**: when you set an issue's org `Department`
field, the issue is added to that team's project board. The issue stays in its
home repo, and no per-repo configuration is needed.

Add it from **Actions → New workflow → "Route issue to team board"**, on the
repo's default branch. Minimal form:

```yaml
name: Route issue to team board

on:
  issues:
    types: [field_added, field_removed]

permissions:
  contents: read

jobs:
  route:
    uses: geolonia/.github/.github/workflows/reusable-route-issue.yml@v1
    secrets: inherit
```

## Options

| Input | Default | Effect |
| --- | --- | --- |
| `exclusive_routing` | `true` | Keep each issue on only its **current** department's board: changing the field moves it to the new board, and clearing it removes it. Set `false` to only ever add (never remove). |

```yaml
jobs:
  route:
    uses: geolonia/.github/.github/workflows/reusable-route-issue.yml@v1
    secrets: inherit
    with:
      exclusive_routing: false   # add only, never remove
```

## Requirements

- The workflow must live on the repo's **default branch**, or the `issues`
  trigger won't fire.
- The repo needs access to the org's shared dispatch secrets (org-level Actions
  secrets). Repos created from the standard scaffolder get this automatically.

## Troubleshooting

Issue field set but nothing appears on the board:

1. Confirm the workflow is on the **default branch** and that its run succeeded
   (Actions tab).
2. The board for that department must exist. Setting a department that has no
   board yet is logged and skipped, not an error.

See the general [Troubleshooting](../workflows.md#troubleshooting) section for
log-fetching tips.
