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
| `exclusive_routing` | `true` | Keep each issue on only the board for its **current** `Department`. Changing the department moves the issue off the other department boards; clearing it removes the issue from all of them. Set `false` for additive routing (add only, never remove). |

```yaml
jobs:
  route:
    uses: geolonia/.github/.github/workflows/reusable-route-issue.yml@v1
    secrets: inherit
    with:
      exclusive_routing: false   # add only, never remove
```

With the default (`exclusive_routing: true`), fixing a mis-set department also
cleans up: picking the wrong option then correcting it leaves the issue on only
the corrected board, not both.

**Only department boards are ever touched.** Removal is scoped strictly to the
projects listed under `departments` in `geolonia-operations`' `routing.yml`. Any
other project an issue happens to sit on is never removed from.

One residual edge: if the department is changed to an option that has **no
board yet** (unmapped), the issue is left where it is and a warning is logged
(there is no destination to move it to).

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
