# Reusable GitHub Actions workflows

This repo hosts org-wide reusable workflows. The pattern uses **two
locations** in this repo plus a **tag-pinning convention** in callers.

## The three roles

| Role | Path | Purpose |
|---|---|---|
| **Template** | `workflow-templates/<name>.yml` + `<name>.properties.json` | Shown in GitHub's "New workflow" picker. Points at a tagged reusable workflow. |
| **Entrypoint** | `.github/workflows/<name>.yml` | This repo's own use of the workflow (push/tag triggers). Forwards to the reusable workflow with defaults. |
| **Reusable** | `.github/workflows/reusable-<name>.yml` | The actual logic. Callable from any repo via `uses: geolonia/.github/.github/workflows/reusable-<name>.yml@<tag>`. |

## Currently maintained workflows

- **`publish-techdocs`** - TechDocs publish on `docs/**` or `mkdocs.yml`
  changes. Uses OIDC for AWS, with `TECHDOCS_AWS_ACCOUNT_ID` org secret
  by default; per-repo `AWS_ACCOUNT_ID` override supported.
- **`release-auto-on-tag`** - semver-tag-triggered releases (`v*.*.*`).
  Optional inputs control prereleases, release notes, custom regex.
- **`cdk-deploy-monitor`** - runs alongside a CDK deploy job, polls
  CloudFormation/CloudWatch/ECS, asks GitHub Models (GPT-4o) whether the
  deploy is hung, optionally cancels on CANCEL verdict. Requires
  `CdkDeployMonitor` IAM permission bundle and `models: read` permission
  in the calling workflow + every parent in the call chain.
- **`sync-team-access`** - keeps GitHub team-to-repo access in sync.

## Caller-side conventions

Consumers should pin to a tag, not a SHA, not `@main`:

```yaml
jobs:
  publish:
    uses: geolonia/.github/.github/workflows/reusable-backstage-techdocs.yml@v1
    secrets: inherit
```

## Updating a reusable workflow

When making **non-breaking** changes:

1. Edit `.github/workflows/reusable-<name>.yml`.
2. Open a PR. Wait for CodeRabbit + human approval.
3. Merge to `main`.
4. Move the `v1` tag forward (or whatever major tag is active):

   ```sh
   git tag -f v1
   git push -f origin v1
   ```

When making **breaking** changes:

1. Cut a new major tag (`v2`, `v3`, …) so consumers upgrade on their own
   schedule.
2. Update `workflow-templates/<name>.yml` to reference the new tag - but
   only after enough consumers have migrated, otherwise new "New workflow"
   selections will land on the latest tag and consumers expecting `v1`
   stay on `v1`.
3. Document the breaking change in a release note on the new tag.

Always keep `workflow-templates/<name>.properties.json` in sync with the
YAML so the picker shows accurate names/descriptions.

## Common troubleshooting

- **`Could not find reusable workflow`** - caller is pinned to a tag that
  doesn't exist (or was deleted). Check
  `gh api repos/geolonia/.github/git/refs/tags --jq '.[].ref'`.
- **`Input required and not supplied`** for AWS account ID - confirm the
  org-level `TECHDOCS_AWS_ACCOUNT_ID` secret is visible to the calling
  repo, or the per-repo `AWS_ACCOUNT_ID` override is set.
- **CDK deploy monitor doesn't post comments** - needs `contents: write`
  + `models: read` on the **caller's** top-level `permissions:` block.
