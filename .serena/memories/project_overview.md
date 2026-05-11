# Project overview

**Name**: `geolonia/.github`
**Owner**: `group:geolonia/operations`
**Visibility**: **Public - must remain public**. GitHub gives this repo
special treatment (see "What lives here") *only* when it is named `.github`
and is publicly accessible.

## Purpose

Single source of truth for organization-wide GitHub assets and policies:

- Org profile content shown on `https://github.com/geolonia` (`profile/`)
- Community health files: `CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`,
  `SUPPORT.md`, `SECURITY.md`, `LICENSE`
- Issue and PR templates (`.github/ISSUE_TEMPLATE/`, `.github/pull_request_template.md`)
- Reusable GitHub Actions workflows (in `.github/workflows/`) and their
  template wrappers (in `workflow-templates/`)
- Org-wide agent policy (`docs/agent-policy.md`) - referenced by every
  Geolonia repo's `AGENTS.md`
- Canonical CodeRabbit configuration (`.coderabbit.yaml`) - referenced by
  every other Geolonia repo via `remote_config:`. See the dedicated
  `shared_coderabbit_config` memory.

## Critical constraints

- **Public-only content**: never commit secrets, internal hostnames,
  customer data, references to private repos, or anything that would
  leak by being world-readable. See the dedicated
  `public_repo_hygiene` memory for the do-not-include checklist.
- **No em-dashes in any commit message, PR body, or markdown**
  (per this repo's `AGENTS.md`).
- **Edits propagate org-wide**: a change here can affect every Geolonia
  repo's PR/issue forms, default templates, and CodeRabbit settings.
  Treat changes here as higher-stakes than typical repo edits.

## Authoritative agent guidance

- This repo's `AGENTS.md` (root) - repo-local rules.
- `docs/agent-policy.md` (this repo) - the org-wide policy that every
  other Geolonia repo's `AGENTS.md` defers to.
