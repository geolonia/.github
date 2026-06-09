# CDK Deploy Monitor (`reusable-cdk-deploy-monitor.yml`)

Runs as a parallel job alongside a CDK deploy. On every poll cycle it fetches the
CloudFormation stack status, recent CloudFormation events, recent CloudWatch logs, and
recently stopped ECS task diagnostics (exit codes and stopped reasons), then asks the
GitHub Models AI (GPT-4o) whether the deployment is progressing normally or is stuck.
The AI verdict is logged to the job output; a CANCEL verdict is also posted as a commit
comment. If `auto_cancel` is enabled and the AI says CANCEL, the workflow calls
`cancel-update-stack`, triggering a rollback and causing the deploy job to fail cleanly
instead of timing out.

## When to use it

Add this to any repo that deploys CDK stacks via GitHub Actions and has experienced
GH Actions timeouts due to hanging CloudFormation updates.

## Prerequisites

- The Geolonia GitHub org has GitHub Copilot enabled (required for GitHub Models API access).
- The calling workflow and all parent workflows in the call chain must declare
  `models: read` in their top-level `permissions:` block.

## Required IAM permissions for `aws_role_arn`

The OIDC role passed as `aws_role_arn` must include the `CdkDeployMonitor` permission
bundle (defined in `geolonia-infra-cdk`), which covers both monitoring and cancellation:

```json
{
  "Effect": "Allow",
  "Action": [
    "cloudformation:DescribeStackEvents",
    "cloudformation:DescribeStacks",
    "cloudformation:DescribeStackResources",
    "cloudformation:CancelUpdateStack",
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

For repos using `geolonia-infra-cdk` to manage their deploy role, add
`permissionBundles: ['CdkDeployMonitor']` to the role entry in the account config.

## Example integration

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

## Input reference

| Input | Type | Default | Description |
|---|---|---|---|
| `stack_name` | string | required | CloudFormation stack name to monitor |
| `aws_region` | string | required | AWS region |
| `log_group_name` | string | `""` | CloudWatch log group name or prefix (leave empty to skip) |
| `hang_threshold_minutes` | number | `10` | Hint to the AI: treat N minutes with no CF events as suspicious |
| `poll_interval_seconds` | number | `120` | How often to poll and run AI analysis (seconds) |
| `auto_cancel` | boolean | `false` | If true, AI verdict of CANCEL triggers `cancel-update-stack` |
| `environment_name` | string | `""` | GitHub Actions environment for environment-scoped secrets |

## `auto_cancel` trade-offs

| Setting | Behaviour |
|---|---|
| `false` (default) | Advisory mode: AI posts a commit comment on CANCEL but never cancels. Safe for initial rollout. |
| `true` | Automatic: cancels the stack on AI CANCEL verdict, stopping the timeout. Requires `CdkDeployMonitorCancel` IAM bundle. |

Start with `auto_cancel: false` to validate AI judgements over a few real deploys before enabling `true`.

## ECS task diagnostics

The monitor automatically looks up the ECS service belonging to the monitored CloudFormation
stack using `cloudformation:DescribeStackResources`. On each poll cycle it also fetches
recently stopped ECS tasks (exit codes and stopped reasons) and includes them in the AI
prompt. This lets the AI explain *why* a deployment is stuck — for example a missing
environment variable, an OOM kill, or a failing health check — rather than just reporting
that no CloudFormation events have occurred.

No extra configuration is needed. The monitor silently skips ECS diagnostics if the stack
contains no ECS service resource.

## Log sensitivity warning

**Do not set `log_group_name`** if your application logs may contain secrets, credentials,
PII, or other sensitive data. Log content is sent to the GitHub Models API (Azure-hosted
OpenAI service) and is also visible in the commit comment. When in doubt, omit the input.

The same applies to ECS stopped task reasons — if task environment variables or startup
errors may reveal sensitive information, be aware that this data is also sent to the AI.

## Troubleshooting the deploy monitor

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
