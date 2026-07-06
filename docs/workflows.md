# Reusable Workflow Templates

Workflow templates in `workflow-templates/` appear in GitHub’s “New workflow”
picker for Geolonia repos. They call the reusable workflows hosted in this repo
under `.github/workflows/`.

## Available workflows

| Template | Page |
| --- | --- |
| Security Suite (`security-suite.yml`) | [Security Suite](workflows/security-suite.md) |
| Publish TechDocs (`publish-techdocs.yml`) | [Publish TechDocs](workflows/publish-techdocs.md) |
| Release on Tag (`release-auto-on-tag.yml`) | [Release on Tag](workflows/release-on-tag.md) |
| Bumblebee Supply-Chain Scan (`bumblebee-scan.yml`) | [Bumblebee Supply-Chain Scan](workflows/bumblebee-scan.md) |
| Action Pinning Check (`pinact-check.yml`) | [Action Pinning Check](workflows/pinact-check.md) |
| CDK Deploy Monitor (`cdk-deploy-monitor.yml`) | [CDK Deploy Monitor](workflows/cdk-deploy-monitor.md) |
| Route issue to team board (`route-issue.yml`) | [Route issue to team board](workflows/route-issue.md) |

## Why two workflow locations?

GitHub displays templates in `workflow-templates/` to users, but the logic lives
in `.github/workflows/`. Templates point to tagged reusable workflows so that
repos making use of them can pin a version and avoid breakage from changes on `main`.

| Role | Example | Purpose |
| --- | --- | --- |
| Template | `workflow-templates/publish-techdocs.yml` | Shown in “New workflow”; references a tagged reusable workflow (e.g., `.../reusable-backstage-techdocs.yml@v1`). |
| Entrypoint | `.github/workflows/publish-techdocs.yml` | Listens to triggers (push, tag) and forwards to the reusable workflow with defaults. |
| Reusable | `.github/workflows/reusable-backstage-techdocs.yml` | Contains the logic; callable from other repos pinned by tag (`@v1`, SHA). |

Pinning to tags prevents breakage in repos making use of them. When a breaking
change is needed, publish a new tag (e.g., `v2.1.0`) so repositories can upgrade
on their own schedule.

> **Security Suite gotcha:** `security-suite.yml` is enforced org-wide as a
> ruleset "required workflow". Its reusable refs may float on `@v1`, but the
> `v1` **tag itself must be lightweight** (an annotated `v1` makes the required
> check hang silently on every consumer PR). See
> [Security Suite](workflows/security-suite.md) before releasing.

## Updating templates

- Keep the `.properties.json` metadata in sync with the YAML so the workflow
  picker shows accurate names/descriptions.
- Bump the referenced reusable workflow version (`@v1`, SHA, or tag) when making
  breaking changes.

## Troubleshooting

For troubleshooting specific workflows, see the per-workflow pages above. The
following tips apply to any reusable workflow in this repo.

### Viewing run logs

Open the Actions tab in the affected repository and click the failed run.
Each job step is expandable — look for the first red step and expand it to
see the full error output.

To fetch logs from the CLI:

```bash
gh run list --workflow=<workflow-file>.yml --limit 5
gh run view <run-id> --log-failed
```

### Triggering a workflow manually

Workflows with a `workflow_dispatch` trigger can be run from the CLI or UI:

```bash
gh workflow run <workflow-file>.yml --ref main
```

From the UI: **Actions → select the workflow → Run workflow**.

### Secrets not found (`Error: Input required and not supplied`)

- Confirm the secret is set under **Settings → Secrets and variables → Actions**
  in the calling repository.
- Reusable workflows called with `secrets: inherit` receive all secrets from the
  caller. If a secret is set at the org level, the calling repo must have access
  to it (check org secret visibility).
- For workflows that override `AWS_ACCOUNT_ID`, make sure the repo-level secret
  name matches exactly (it is case-sensitive).

### Reusable workflow not found (`Could not find reusable workflow`)

The caller is likely pinned to a tag that does not exist yet in this repo, or
the ref has been deleted. Check whether the tag exists directly:

```bash
git ls-remote --tags https://github.com/geolonia/.github | grep <tag-name>
# or via the GitHub API:
gh api repos/geolonia/.github/git/refs/tags --jq '.[].ref' | grep <tag-name>
```

As a quick fallback you can also list releases (note: tags and releases are not
always in sync):

```bash
gh release list --repo geolonia/.github --limit 10
```

If the tag is missing, create it from the appropriate commit on `main`.
