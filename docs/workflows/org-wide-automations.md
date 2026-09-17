# Org-wide automations (issue routing, team access sync)

Two automations reach every repository of the `geolonia` organization without a
workflow file in the repository. The GitHub Apps installed on the organization
deliver their `issues` and `push` webhooks to an endpoint run by the operations
team, which forwards the relevant ones to the central automation in
`geolonia-operations`. A repository takes part through a **repository custom
property**.

| Automation | Property | What happens when it is `true` |
| --- | --- | --- |
| Route issue to team board | `issue-routing` | When an issue's org `Department` field is set, changed or cleared, the issue is added to (or removed from) that team's project board. |
| Sync Team Access | `team-access-sync` | When `catalog-info.yaml` changes on the default branch, the central team-access sync reconciles the repository's access within a minute instead of at the nightly run. |

## Opting a repository in

Custom property values can be edited by organization owners and by the
operations team, not by repository admins. Ask the operations team, or set the
value yourself if you have the permission:

```bash
gh api -X PATCH repos/geolonia/<repo>/properties/values \
  --input - <<'JSON'
{"properties":[{"property_name":"issue-routing","value":"true"},
               {"property_name":"team-access-sync","value":"true"}]}
JSON
```

Repositories created through the standard scaffolder get the properties at
creation time. Set a property to `false` or clear it to opt out again.

## What the repository does not need any more

- No `route-issue.yml` or `sync-team-access.yml` workflow. Repositories that
  still carry one produce a duplicate run of the central automation until the
  file is removed; the result is the same, so removal is not urgent, but do not
  add the file to new repositories.
- No access to the shared dispatch secrets. The forwarding credential lives
  with the endpoint, not in the repositories.

## How it stays safe

The forwarder only says "something changed in repository X". The central
automation re-reads the `Department` field and `catalog-info.yaml` from GitHub
itself, so nothing in a webhook payload can route an issue or grant access on
its own. Requests to the endpoint are authenticated with the webhook secret
GitHub signs every delivery with.

## Troubleshooting

An issue field was set but nothing appears on the board:

1. Check the repository's custom properties (**Settings**, then **Custom
   properties**, or `gh api repos/geolonia/<repo>/properties/values`) for
   `issue-routing` = `true`.
2. The board for that department must exist. Setting a department that has no
   board yet is logged and skipped by the central automation, not an error.
3. Ask the operations team to check the forwarder's log for the delivery; every
   delivery is logged with the reason it was forwarded or ignored.

A `catalog-info.yaml` change was merged but access did not update:

1. Check `team-access-sync` = `true` on the repository.
2. The change must have reached the **default branch**; pushes to other
   branches are ignored.
3. In report mode the sync reports without applying; see the run in
   `geolonia-operations` (Actions, then **Team Access Sync**).
