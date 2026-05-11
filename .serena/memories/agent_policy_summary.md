# Org-wide agent policy summary

The authoritative document is `docs/agent-policy.md` in this repo. Every
Geolonia repo's `AGENTS.md` references it. Repo-specific rules may extend
it but should not conflict with it.

Quick summary so future sessions can decide whether to read the full doc:

## Issue-first workflow (org-wide rule)

- **Required** for feature/bugfix work that affects user-facing
  behaviour or touches multiple files.
- **Optional** for docs, typos, and small single-file refactors.
- Branch naming: `issue-<n>-<slug>` from `main`. (Note: this differs from
  the `chore/<topic>` / `feat/<topic>` patterns used in many submodules -
  the org policy is more flexible; submodule-specific patterns win in
  practice.)
- Manual merge after checks pass. No auto-merge.

This is the org-wide source of the "issue-first" pattern that the global
memory `global/issue-first-pr-workflow` builds on.

## Code reviews

- CodeRabbit reviews every PR within minutes. Wait for it before
  requesting a human reviewer.
- Address every CodeRabbit comment. Resolve each thread after applying
  the fix or replying with a rationale for declining.
- Don't merge until: (1) CI passes, (2) all CodeRabbit blocking comments
  are resolved, (3) at least one human reviewer has approved.

## Team communication

- **Don't use Slack** for ops requests / escalations. File a GitHub issue
  in the relevant repo and assign it. Slack is informal-only.

## Safety / hygiene

- Never commit secrets or private keys.
- Avoid destructive `git` commands unless explicitly requested.
- Keep changes minimal and aligned with existing patterns.

## Related org docs

- `docs/context-maintenance.md`
- `docs/roadmap-workflow.md`
- `docs/tooling.md`
