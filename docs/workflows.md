# Reusable Workflow Templates

Workflow templates in `workflow-templates/` appear in GitHub’s “New workflow”
picker for Geolonia repos. They call the reusable workflows hosted in this repo
under `.github/workflows/`.

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

## Publish TechDocs (`publish-techdocs.yml`)

- Runs on `main` when `docs/**` or `mkdocs.yml` changes, or manually via `workflow_dispatch`.
- Delegates to `reusable-backstage-techdocs.yml@v1` with safe defaults for AWS
  region, environment, and tool versions.
- Uses `TECHDOCS_AWS_ACCOUNT_ID` of `geolonia/.github` repository secret by default;
  optional inputs allow setting a repo secret `AWS_ACCOUNT_ID` to override on a
  per-repo basis.

Example minimal usage after selecting the template:

```yaml
jobs:
  publish:
    uses: geolonia/.github/.github/workflows/reusable-backstage-techdocs.yml@v1
    # Optional repo override:
    # with:
    #   environment: production
    #   aws_region: ap-northeast-1
    secrets: inherit
    # In case you want to override the AWS account ID:
    # secrets: 
    #   AWS_ACCOUNT_ID: ${{ secrets.TECHDOCS_AWS_ACCOUNT_ID }}
```

## Release on Tag (`release-auto-on-tag.yml`)

- Triggers on semver-like tags (`v*.*.*`).
- Delegates to `reusable-release-auto-on-tag.yml@v1` to create releases and move
  major/minor tags if configured.
- Inherits calling-repo secrets; optional inputs allow toggling prereleases,
  generating release notes, or customizing the tag regex.

Example minimal usage:

```yaml
jobs:
  publish:
    uses: geolonia/.github/.github/workflows/reusable-release-auto-on-tag.yml@v1
    secrets: inherit
```

## Updating templates

- Keep the `.properties.json` metadata in sync with the YAML so the workflow
  picker shows accurate names/descriptions.
- Bump the referenced reusable workflow version (`@v1`, SHA, or tag) when making
  breaking changes.

## Troubleshooting

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

### TechDocs publish fails with AWS credentials error

Ensure the workflow environment has access to the OIDC-based credentials.
Check that the AWS account ID secret (`TECHDOCS_AWS_ACCOUNT_ID` or the repo
override `AWS_ACCOUNT_ID`) is set and non-empty, and that the IAM role trust
policy allows the `geolonia` GitHub org.

## CDK Deploy Monitor (`reusable-cdk-deploy-monitor.yml`)

Runs as a parallel job alongside a CDK deploy to detect hanging CloudFormation stacks.
When no new CloudFormation events appear for a configurable threshold, it fetches
CloudWatch logs and asks the GitHub Models AI (GPT-4o) whether the deployment is stuck
or just slow. The verdict is posted as a commit comment. If `auto_cancel` is enabled and
the AI says CANCEL, the workflow calls `cancel-update-stack`, triggering a rollback and
causing the deploy job to fail cleanly instead of timing out.

### When to use it

Add this to any repo that deploys CDK stacks via GitHub Actions and has experienced
GH Actions timeouts due to hanging CloudFormation updates.

### Required IAM permissions for `aws_role_arn`

The OIDC role passed as `aws_role_arn` must include the `CdkDeployMonitor` permission
bundle (defined in `geolonia-infra-cdk`):

```json
{
  "Effect": "Allow",
  "Action": [
    "cloudformation:DescribeStackEvents",
    "cloudformation:DescribeStacks",
    "cloudformation:CancelUpdateStack",
    "logs:DescribeLogGroups",
    "logs:DescribeLogStreams",
    "logs:FilterLogEvents",
    "logs:GetLogEvents"
  ],
  "Resource": "*"
}
```

For repos using `geolonia-infra-cdk` to manage their deploy role, add
`permissionBundles: ['CdkDeployMonitor']` to the role entry in the account config.

### Example integration

```yaml
jobs:
  deploy:
    # ... your existing deploy job, unchanged

  monitor:
    uses: geolonia/.github/.github/workflows/reusable-cdk-deploy-monitor.yml@v1
    with:
      stack_name: MyAppStack
      aws_region: ap-northeast-1
      log_group_name: "MyAppStack-MyLogGroup"   # optional
      hang_threshold_minutes: 10                 # optional, default: 10
      auto_cancel: false                         # optional, default: false
    secrets:
      aws_role_arn: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/YOUR_DEPLOY_ROLE
```

Also add `contents: write` and `pull-requests: write` to the calling workflow's top-level
`permissions:` block. The monitor uses the commit comments API (requires `contents: write`)
and may annotate pull requests (requires `pull-requests: write`).

### Input reference

| Input | Type | Default | Description |
|---|---|---|---|
| `stack_name` | string | required | CloudFormation stack name to monitor |
| `aws_region` | string | required | AWS region |
| `log_group_name` | string | `""` | CloudWatch log group name or prefix (leave empty to skip) |
| `hang_threshold_minutes` | number | `10` | Minutes of silence before asking AI |
| `poll_interval_seconds` | number | `120` | How often to poll (seconds) |
| `auto_cancel` | boolean | `false` | If true, AI verdict of CANCEL triggers `cancel-update-stack` |

### `auto_cancel` trade-offs

| Setting | Behaviour |
|---|---|
| `false` (default) | Advisory mode: AI posts a comment but never cancels. Safe for initial rollout. |
| `true` | Automatic: cancels the stack on AI verdict, stopping the timeout. Preferred once you trust the AI accuracy. |

Start with `auto_cancel: false` to validate AI judgements over a few real deploys before enabling `true`.

### Log sensitivity warning

**Do not set `log_group_name`** if your application logs may contain secrets, credentials,
PII, or other sensitive data. Log content is sent to the GitHub Models API (Azure-hosted
OpenAI service) and is also visible in the commit comment. When in doubt, omit the input.

### Troubleshooting the deploy monitor

**Monitor exits immediately without analysing:**
The stack may have already reached a terminal state before the first poll. This is normal
for fast deploys. Check the `Monitor stack` step log for the final status line.

**AI analysis not firing:**
Confirm `hang_threshold_minutes` is not set too high relative to your GH Actions job
timeout. Check the `Monitor stack` step logs for `[HH:MM:SS] Stack status:` lines.

**Commit comment not posted:**
Ensure the calling workflow has `contents: write` in its `permissions:` block (commit
comments require write access to repository contents). Also add `pull-requests: write`
if you want AI verdicts to appear on pull request timelines.

**`cancel-update-stack` returns an error:**
The stack may have already reached a terminal state between the AI analysis and the
cancel call. This is harmless. Check the commit comment for the error detail.
The OIDC role must also have `cloudformation:CancelUpdateStack` -- verify the
`CdkDeployMonitor` permission bundle is attached.
