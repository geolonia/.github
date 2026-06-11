# Route issue to team board (`route-issue.yml`)

Opt a repo in to **org-wide issue auto-routing**: when an issue's org-level
`Department` issue field is set, the issue is added to the matching team
project board (Operations, Security, … Design later). The issue stays in its
home repo and can sit on multiple boards at once.

- Runs on `issues: [field_added, field_removed]` in the participating repo.
- Delegates to `reusable-route-issue.yml@v1`, which forwards the event to the
  private `geolonia-operations` repo via `repository_dispatch`.
- No per-repo configuration. The routing table (`Department → project`) and the
  privileged Org-Projects token live **only** in `geolonia-operations`.

Example minimal usage after selecting the template:

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
| `remove_on_field_removed` | `false` | When the issue's `Department` field is **cleared**, also remove the issue from its team board(s). Default keeps it on the board. |

```yaml
jobs:
  route:
    uses: geolonia/.github/.github/workflows/reusable-route-issue.yml@v1
    secrets: inherit
    with:
      remove_on_field_removed: true
```

Removal only fires when the field is **cleared**. Changing `Department` from one
option to another fires `field_added` (an update), not `field_removed`, so a
move between boards adds to the new board without removing the old item. A
configured master board is never removed from.

## Why dispatch-to-central?

This public side is a dumb forwarder. It mints only a low-privilege dispatch
token and tells `geolonia-operations` "issue node id X changed". It never reads
the field value, never sees a project id, and never holds the Org-Projects
credential. That keeps every secret and routing decision private, mirroring the
[Sync Team Access](../workflows.md) dispatch-to-central design (individual repos
never see the org token).

```text
participating repo            geolonia/.github (public)        geolonia-operations (private)
route-issue.yml  ──uses──▶  reusable-route-issue.yml
 on: issues:                  mint dispatch token
  [field_added,               repository_dispatch ──────────▶  process-route-issue.yml
   field_removed]                                              read Department → add to board
```

## Prerequisites

- The repo must be able to see the org dispatch secrets
  `OPS_DISPATCH_CLIENT_ID` and `OPS_DISPATCH_APP_PRIVATE_KEY` (org-level Actions
  secrets, already used by Sync Team Access).
- The caller workflow must live on the repo's **default branch**, otherwise the
  `issues` trigger does not fire.

## Troubleshooting

### Issue field set but nothing lands on the board

1. Confirm the caller workflow is on the **default branch** and ran (Actions tab
   of the participating repo).
2. Confirm the dispatch step succeeded and check that the matching
   `repository_dispatch` run started in `geolonia-operations`
   (**Actions → Route Issue to Team Board**).
3. Confirm the `Department` option name exists in `geolonia-operations`'
   `scripts/route-issue/routing.yml`. An unmapped department is logged and
   skipped, not an error.

### Dispatch step fails with a token/permission error

The org dispatch secrets must be visible to the calling repo (check org secret
visibility) and the dispatch app must be installed on `geolonia-operations`.

See also the general [Troubleshooting](../workflows.md#troubleshooting) section
for log-fetching tips.

> Issue fields are in **public preview**. The single field-value read is
> isolated in `geolonia-operations`' `scripts/route-issue/route.sh`, so a
> preview API change is a one-line edit there with no impact on participating
> repos.
