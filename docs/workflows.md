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

Runs as a parallel job alongside a CDK deploy. On every poll cycle it fetches the
CloudFormation stack status, recent CloudFormation events, recent CloudWatch logs, and
recently stopped ECS task diagnostics (exit codes and stopped reasons), then asks the
GitHub Models AI (GPT-4o) whether the deployment is progressing normally or is stuck.
The AI verdict is logged to the job output; a CANCEL verdict is also posted as a commit
comment. If `auto_cancel` is enabled and the AI says CANCEL, the workflow calls
`cancel-update-stack`, triggering a rollback and causing the deploy job to fail cleanly
instead of timing out.

### When to use it

Add this to any repo that deploys CDK stacks via GitHub Actions and has experienced
GH Actions timeouts due to hanging CloudFormation updates.

### Prerequisites

- The Geolonia GitHub org has GitHub Copilot enabled (required for GitHub Models API access).
- The calling workflow and all parent workflows in the call chain must declare
  `models: read` in their top-level `permissions:` block.

### Required IAM permissions for `aws_role_arn`

The OIDC role passed as `aws_role_arn` must include the `CdkDeployMonitor` permission
bundle (defined in `geolonia-infra-cdk`):

```json
{
  "Effect": "Allow",
  "Action": [
    "cloudformation:DescribeStackEvents",
    "cloudformation:DescribeStacks",
    "cloudformation:DescribeStackResources",
    "logs:DescribeLogGroups",
    "logs:DescribeLogStreams",
    "logs:FilterLogEvents",
    "logs:GetLogEvents",
    "ecs:ListTasks",
    "ecs:DescribeTasks"
  ],
  "Resource": "*"
}
```

To enable `auto_cancel: true`, also attach the `CdkDeployMonitorCancel` bundle:

```json
{
  "Effect": "Allow",
  "Action": ["cloudformation:CancelUpdateStack"],
  "Resource": "*"
}
```

For repos using `geolonia-infra-cdk` to manage their deploy role, add
`permissionBundles: ['CdkDeployMonitor']` (and `'CdkDeployMonitorCancel'` when using
`auto_cancel: true`) to the role entry in the account config.

### Example integration

```yaml
# Top-level permissions must include all permissions the monitor needs.
# If this workflow is called as a reusable workflow, the parent must also
# declare the same permissions.
permissions:
  id-token: write
  contents: write
  models: read

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
      aws_role_arn: ${{ needs.build-and-push.outputs.monitor_role_arn }}
```

> **Note on passing the role ARN:** The `secrets:` block of a reusable workflow call
> does not have access to the `env` context. The recommended pattern is to construct the
> full ARN in a preceding job that has `environment: production` (where `vars.AWS_ACCOUNT_ID`
> resolves correctly), expose it as a job output, and pass it via `needs.<job>.outputs.<key>`.
>
> ```yaml
> jobs:
>   build-and-push:
>     environment: production
>     outputs:
>       monitor_role_arn: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/github-actions-cdk-deploy-myapp
>     steps: [...]
>
>   monitor:
>     needs: build-and-push
>     uses: geolonia/.github/.github/workflows/reusable-cdk-deploy-monitor.yml@v1
>     secrets:
>       aws_role_arn: ${{ needs.build-and-push.outputs.monitor_role_arn }}
> ```

### Input reference

| Input | Type | Default | Description |
|---|---|---|---|
| `stack_name` | string | required | CloudFormation stack name to monitor |
| `aws_region` | string | required | AWS region |
| `log_group_name` | string | `""` | CloudWatch log group name or prefix (leave empty to skip) |
| `hang_threshold_minutes` | number | `10` | Hint to the AI: treat N minutes with no CF events as suspicious |
| `poll_interval_seconds` | number | `120` | How often to poll and run AI analysis (seconds) |
| `auto_cancel` | boolean | `false` | If true, AI verdict of CANCEL triggers `cancel-update-stack` |
| `environment_name` | string | `""` | GitHub Actions environment for environment-scoped secrets |

### `auto_cancel` trade-offs

| Setting | Behaviour |
|---|---|
| `false` (default) | Advisory mode: AI posts a commit comment on CANCEL but never cancels. Safe for initial rollout. |
| `true` | Automatic: cancels the stack on AI CANCEL verdict, stopping the timeout. Requires `CdkDeployMonitorCancel` IAM bundle. |

Start with `auto_cancel: false` to validate AI judgements over a few real deploys before enabling `true`.

### ECS task diagnostics

The monitor automatically looks up the ECS service belonging to the monitored CloudFormation
stack using `cloudformation:DescribeStackResources`. On each poll cycle it also fetches
recently stopped ECS tasks (exit codes and stopped reasons) and includes them in the AI
prompt. This lets the AI explain *why* a deployment is stuck — for example a missing
environment variable, an OOM kill, or a failing health check — rather than just reporting
that no CloudFormation events have occurred.

No extra configuration is needed. The monitor silently skips ECS diagnostics if the stack
contains no ECS service resource.

### Log sensitivity warning

**Do not set `log_group_name`** if your application logs may contain secrets, credentials,
PII, or other sensitive data. Log content is sent to the GitHub Models API (Azure-hosted
OpenAI service) and is also visible in the commit comment. When in doubt, omit the input.

The same applies to ECS stopped task reasons — if task environment variables or startup
errors may reveal sensitive information, be aware that this data is also sent to the AI.

### Troubleshooting the deploy monitor

**Monitor exits immediately on startup:**
If the stack was already in a terminal state (e.g. from a previous run), the monitor
waits up to 5 minutes for the deploy to begin. If CDK has not started the CloudFormation
update within that window, the monitor exits. Check the `Monitor and analyse` step log
for `Initial stack status:` to confirm.

**AI shows as unreachable:**
Confirm the org has GitHub Copilot enabled and that `models: read` is declared in the
calling workflow's top-level `permissions:` block. When the AI is unreachable, the
monitor defaults to CONTINUE and keeps polling.

**Commit comment not posted:**
Ensure the calling workflow has `contents: write` in its top-level `permissions:` block.
Comments are only posted on CANCEL verdicts. CONTINUE verdicts are only logged to the
job output.

**`cancel-update-stack` fails:**
The stack may have already reached a terminal state between the AI verdict and the cancel
call -- this is harmless. Check the commit comment for the error detail. Also verify the
OIDC role includes `CancelUpdateStack` via the `CdkDeployMonitorCancel` permission bundle.
