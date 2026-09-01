# Agent Policy

This document defines organization-wide guidance for AI agents working in Geolonia
repositories. Repository-specific rules may extend this policy but should not
conflict with it.

Related org docs:

- `docs/context-maintenance.md`
- `docs/roadmap-workflow.md`
- `docs/tooling.md`

## Issue-first workflow

- Required for feature/bugfix work that affects user-facing behavior or touches
  multiple files.
- Optional for docs, typos, and small single-file refactors.
- Use the repo issue template if present.
- Branch naming: `issue-<n>-<slug>` from `main`.
- Manual merge after checks pass.

Recommended command flow:

```bash
gh issue create --template issue-first.md --label "<label>"
git checkout -b issue-<n>-<slug>
pnpm lint && pnpm test
gh pr create --title "<title>" --body "Fixes #<n>" --label "<label>"
```

## Labels

- Apply labels on issues and PRs (use existing defaults like `bug`,
  `enhancement`, `documentation`).

## Safety and hygiene

- Never commit secrets or private keys.
- Avoid destructive git commands unless explicitly requested.
- Keep changes minimal and aligned with existing patterns.

## CI and GitHub Actions security

These rules apply to every workflow you add or edit. The `zizmor` job in the
Security Suite gates pull requests on error-severity findings, so a workflow
that breaks them will usually fail the check before a human looks at it.
`pinact` findings are warn-only and do not block a merge.

- **Default to `pull_request`.** For a pull request from a fork it runs with a
  read-only `GITHUB_TOKEN` and no repository secrets, so long as the repository
  has not opted into sending write tokens or secrets to fork pull request
  workflows. A private repository can enable both in its Actions settings, so
  confirm that before relying on the guarantee.
- **Never run untrusted pull request code with secrets in scope.**
  `pull_request_target` and `workflow_run` run in the context of the base
  repository, with its secrets and a write-capable token. Combining either with
  a checkout of the pull request head (`github.event.pull_request.head.sha`, or
  the head ref) and any step that executes repository content (a build, a test,
  an install that runs lifecycle scripts) hands that token to whoever opened the
  pull request. See
  [GitHub's guidance on securely using `pull_request_target`](https://docs.github.com/en/actions/reference/security/securely-using-pull_request_target).
- **To read untrusted code and still write to the pull request, use two
  workflows, not two jobs.** `permissions:` can only narrow the token GitHub
  issued for the event, never widen it, so a second job in the same
  fork-triggered `pull_request` run cannot gain write access. Run the untrusted
  code in the `pull_request` workflow and upload its output as an artifact, then
  have a separate `workflow_run` workflow, which runs in the base repository
  context, download that artifact and post the result. The privileged workflow
  must never check out the pull request head, and must treat the artifact as
  untrusted input, because it was produced by a job that ran the pull request's
  own code. Confirm the artifact came from the expected workflow and run,
  validate its shape before using any value from it, and never execute or
  source a file out of it. A pull request number read from an unvalidated
  artifact can point the privileged workflow at a different pull request.
- **Set least-privilege `permissions:`.** Declare them explicitly, per job where
  jobs differ, and grant only what the job uses. An omitted block inherits the
  configured default, which may be broader than the job needs. Use
  `permissions: {}` for a job that needs no repository or API access at all;
  `GITHUB_TOKEN` still exists, it simply carries no scopes.
- **Pin third-party actions to a full commit SHA.** See
  [Pinning GitHub Actions](github-actions-pinning.md).
- **Treat `github.event.*` as untrusted input.** Pull request titles, branch
  names, and bodies are controlled by whoever opened them. Do not interpolate
  them into a `run:` script; pass them through `env:` and quote the variable.
- **Do not persist credentials a job does not need.** Pass
  `persist-credentials: false` to `actions/checkout` when the job performs no
  git operation, so the token is not left in `.git/config` for later steps to
  read.

## Code reviews

- After opening a PR, CodeRabbit posts an automated review within a few minutes.
- Address all CodeRabbit comments. Resolve each thread after applying the fix
  (or leave a reply explaining why the suggestion was declined).
- Human approval is not required by default. CodeRabbit approves the PR once every
  comment it raised is resolved and it has reviewed the latest commit, and that
  approval satisfies the required review.
- Do not merge until:
  1. All CI checks pass.
  2. All CodeRabbit comments are resolved.
  3. The PR has an approving review.
- Request a human reviewer whenever the change deserves human judgement, for
  example security sensitive work, infrastructure, or anything hard to reverse.
  Some changes still require one: a PR touching a path listed in `CODEOWNERS`
  needs an approval from the owning team no matter what CodeRabbit says.
- Address human reviewer feedback promptly and resolve threads when the change is applied.

Two cases produce no CodeRabbit review at all, and therefore no approval:

- **Draft PRs.** Mark the PR ready for review to get one.
- **PRs based on another feature branch.** CodeRabbit only reviews PRs that target
  the default branch. Retarget the PR, or ask a human to review it.

## Team communication

- Do not use Slack to request work from or escalate issues to the operations team.
- Create a GitHub issue in the relevant repository instead and assign it to a team member.
- Slack is for informal, synchronous conversation only; it is not a task queue.
