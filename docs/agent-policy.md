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

## Code reviews

- After opening a PR, CodeRabbit will post an automated review within a few minutes.
  Wait for it before asking a human reviewer for approval.
- Address all CodeRabbit comments. Resolve each thread after applying the fix
  (or leave a reply explaining why the suggestion was declined).
- Do not merge until:
  1. All CI checks pass.
  2. All CodeRabbit blocking comments are resolved.
  3. At least one human reviewer has approved.
- Address human reviewer feedback promptly and resolve threads when the change is applied.

## Team communication

- Do not use Slack to request work from or escalate issues to the operations team.
- Create a GitHub issue in the relevant repository instead and assign it to a team member.
- Slack is for informal, synchronous conversation only; it is not a task queue.
