# CDK Deploy Monitor (`reusable-cdk-deploy-monitor.yml`)

Runs as a parallel job alongside a CDK deploy. On every poll cycle it fetches the
CloudFormation stack status and recent events, the resources still in flight, recently
stopped ECS tasks (exit codes and stopped reasons) and, optionally, recent CloudWatch
logs, then applies a deterministic rule to decide whether the update is progressing or
hung. A CANCEL verdict is posted once as a commit comment (retried on later polls if
GitHub was unreachable). If `auto_cancel` is enabled and the stack is in
`UPDATE_IN_PROGRESS`, the workflow also calls
`cancel-update-stack`, triggering a rollback so the deploy job fails cleanly instead of
sitting in the six-hour Actions timeout.

The verdict used to come from GitHub Models. That service was retired on 2026-07-30, so
the rule below now encodes the same reasoning without an external model.

## When to use it

Add this to any repo that deploys CDK stacks via GitHub Actions and has experienced
GH Actions timeouts due to hanging CloudFormation updates.

## How the verdict is decided

The update's start time is the stack's `LastUpdatedTime` (`CreationTime` for a first
deploy), read once. On each poll the monitor then computes:

- **Minutes since the last CloudFormation event** for the stack.
- **Resources in flight**: every resource whose current status, from
  `describe-stack-resources`, is `*_IN_PROGRESS`. This is read from the resource list, not
  inferred from a page of events, so a busy update cannot hide a slow resource.
- **Abnormally stopped ECS tasks** since the update started: stopped tasks whose reason
  is anything other than being drained by the deployment (`Scaling activity initiated by
  (deployment ...)`). Essential container exits, OOM kills, failed ELB health checks and
  image pull or init errors all count. The whole `ListTasks` page (100 tasks) is
  inspected and filtered by stop time.

A poll on which CloudFormation cannot be read produces no verdict. Three consecutive
failed reads stop the monitor with an error, so an outage or a permissions problem is
never mistaken for a healthy CONTINUE.

Then, in this order:

1. If `max_failed_tasks` is greater than zero and that many tasks have stopped
   abnormally, the verdict is **CANCEL**: the new task definition is not becoming
   healthy and CloudFormation would otherwise wait for ECS to give up on its own.
2. Otherwise, if the stack has been silent for `hang_threshold_minutes`, the verdict is
   **CANCEL**, with one exception: while *every* in-flight resource is one of
   `slow_resource_types` (by default ECS services, CloudFront distributions, ACM
   certificates and RDS instances or clusters, all of which legitimately produce no
   events for a long time), `slow_resource_grace_minutes` replaces the threshold
   outright, whether it is higher or lower.
3. Otherwise the verdict is **CONTINUE**.

Every poll logs the verdict together with the silence so far, the effective threshold,
what is in flight and the abnormal task count, so a CONTINUE is as explainable as a
CANCEL.

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

Declare the monitor's permissions on the monitor job, not at the top level.
`contents: write` at the top level would also widen your deploy job, which
usually needs only `contents: read` for its checkout. If this workflow is itself
called as a reusable workflow, the parent must grant at least these, because a
caller can only narrow the token, never widen it.

```yaml
jobs:
  deploy:
    # ... your existing deploy job, unchanged
    environment: production
    permissions:
      id-token: write
      contents: read # checkout only

  monitor:
    permissions:
      id-token: write # OIDC: assume the monitor role
      contents: write # post deploy-status commit comments
    uses: geolonia/.github/.github/workflows/reusable-cdk-deploy-monitor.yml@v1
    with:
      stack_name: MyAppStack
      aws_region: ap-northeast-1
      log_group_name: "MyAppStack-MyLogGroup"   # optional
      hang_threshold_minutes: 10                 # optional, default: 10
      auto_cancel: false                         # optional, default: false
      environment_name: production               # optional, same gate as the deploy job
    secrets:
      aws_role_arn: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/github-actions-cdk-deploy-myapp
```

> **Note on the account id:** the `secrets:` block of a reusable workflow call is
> evaluated in the calling file's context, which has no `environment`, so an
> environment-scoped secret is empty there. Use a repository **variable** for the
> account id, or build the ARN in a preceding job that declares the environment and
> pass it through `needs.<job>.outputs.<key>`.

`environment_name` makes the monitor job wait on the same environment protection rules
as the deploy job, so a required approval or wait timer cannot let the monitor's five
minute startup window expire before the deploy has started.

## Input reference

| Input | Type | Default | Description |
|---|---|---|---|
| `stack_name` | string | required | CloudFormation stack name to monitor |
| `aws_region` | string | required | AWS region |
| `log_group_name` | string | `""` | CloudWatch log group name or prefix; the last lines are attached to a CANCEL comment (leave empty to skip) |
| `hang_threshold_minutes` | number | `10` | Minutes with no CloudFormation event after which the update is judged hung |
| `slow_resource_grace_minutes` | number | `20` | Replaces `hang_threshold_minutes` (even when lower) while only `slow_resource_types` are in flight |
| `slow_resource_types` | string | ECS service, CloudFront distribution, ACM certificate, RDS instance and cluster | Space-separated CloudFormation resource types that get the longer grace |
| `max_failed_tasks` | number | `3` | CANCEL once this many ECS tasks have stopped abnormally during the update; `0` disables |
| `poll_interval_seconds` | number | `120` | How often to poll and evaluate (seconds) |
| `auto_cancel` | boolean | `false` | If true, a CANCEL verdict on an `UPDATE_IN_PROGRESS` stack triggers `cancel-update-stack` |
| `environment_name` | string | `""` | GitHub Actions environment for the monitor job |

## `auto_cancel` trade-offs

| Setting | Behaviour |
|---|---|
| `false` (default) | Advisory mode: a CANCEL verdict posts one commit comment and the monitor keeps polling. Safe for initial rollout. |
| `true` | Automatic: a CANCEL verdict on an `UPDATE_IN_PROGRESS` stack cancels the update, stopping the timeout. Creates and rollbacks cannot be cancelled and fall back to the advisory comment. |

Start with `auto_cancel: false`, read the per-poll verdict lines over a few real deploys,
and tune `hang_threshold_minutes` or `slow_resource_grace_minutes` for your stack before
enabling `true`. A stack whose updates routinely involve a slow resource type not in the
default list (for example an OpenSearch domain) should add it to `slow_resource_types`.

## ECS task diagnostics

The monitor automatically looks up the ECS service belonging to the monitored CloudFormation
stack using `cloudformation:DescribeStackResources`. On each poll cycle it fetches
recently stopped ECS tasks and uses them twice: to count abnormal stops for the crash-loop
rule, and to list the stopped reasons and exit codes in a CANCEL comment, so the reader
sees *why* the deployment was stuck (a missing environment variable, an OOM kill, a
failing health check) rather than just that it went quiet.

No extra configuration is needed. The monitor silently skips ECS diagnostics if the stack
contains no ECS service resource.

## Log sensitivity warning

**Do not set `log_group_name`** if your application logs may contain secrets, credentials,
PII, or other sensitive data. The last lines are attached to the CANCEL commit comment,
which is visible to everyone who can read the repository. When in doubt, omit the input.

The same applies to ECS stopped task reasons: if startup errors may reveal sensitive
information, be aware that they are quoted in the comment as well.

## Troubleshooting the deploy monitor

**Monitor exits immediately on startup:**
If the stack was already in a terminal state (e.g. from a previous run), the monitor
waits up to 5 minutes for the deploy to begin. If CDK has not started the CloudFormation
update within that window, the monitor exits. Check the `Monitor and analyse` step log
for `Initial stack status:` to confirm. If the deploy job waits on an environment
approval, pass the same environment through `environment_name`.

**A healthy deploy was cancelled:**
Read the `Verdict:` lines in the job log. If the stack was silent because of a slow
resource type that is not in the default list, add it to `slow_resource_types`; if the
whole deploy is simply slow, raise `hang_threshold_minutes`. If tasks were counted as
abnormal stops during a legitimate restart, raise `max_failed_tasks` or set it to `0`.

**Commit comment not posted:**
Ensure the calling workflow has `contents: write` in its `permissions:` block for the
monitor job and for every parent job in the call chain. Comments are only posted on
CANCEL verdicts. CONTINUE verdicts are only logged to the job output. A transient GitHub
failure is retried on the next two polls; if the comment still cannot be posted the job
ends with an error annotation saying so.

**Monitor stops with `Could not read CloudFormation`:**
Three consecutive polls failed to read the stack. Check the OIDC role's
`cloudformation:DescribeStacks`, `DescribeStackEvents` and `DescribeStackResources`
permissions and the region input; the first failed read is logged with the AWS error.

**`cancel-update-stack` fails:**
The stack may have already reached a terminal state between the verdict and the cancel
call, which is harmless. Check the commit comment for the error detail. Also verify the
OIDC role includes `CancelUpdateStack` via the `CdkDeployMonitor` permission bundle.
