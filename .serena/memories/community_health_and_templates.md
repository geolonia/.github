# Community health files and templates

GitHub auto-applies these to every Geolonia repo that doesn't override
them locally. Touching them affects the whole org - treat as a
higher-stakes change.

## Community health (auto-inherited)

| File | Effect |
|---|---|
| `CODE_OF_CONDUCT.md` | Linked from any repo's "Insights → Community" tab; based on Contributor Covenant. |
| `CONTRIBUTING.md` | Shown on PR creation; documents contribution flow + review expectations. |
| `SUPPORT.md` | Linked from any repo's "Issues → Get help" prompt. |
| `SECURITY.md` | Linked from any repo's "Security" tab; vulnerability reporting process. |
| `LICENSE` | Default for new repos until they add their own. |

## Issue / PR templates (auto-inherited)

- `.github/ISSUE_TEMPLATE/bug_report.yml` - labels new issues with `bug`.
- `.github/ISSUE_TEMPLATE/feature_request.yml` - labels new issues with `enhancement`.
- `.github/ISSUE_TEMPLATE/config.yml` - controls "Open a blank issue" link
  and points to commercial support.
- `.github/pull_request_template.md` - prompts for `Fixes #N` linkage,
  summary, optional tests/docs checklist.

## Maintenance rules

- Update this repo first when a guideline changes - every project picks
  it up automatically.
- Keep contact emails / links current in the policy files.
- **Never include repo-specific information** here. If a single project
  needs a different policy, use a per-repo override in that repo
  (commit a local file with the same name).
- **No sensitive information** ever - repo is and must remain public.

## Override behaviour

A repo can override any inherited file by committing a file at the same
path locally. GitHub picks the closest one (repo-local first,
`.github` repo second). This means accidentally creating an
`.github/pull_request_template.md` in another repo silently shadows the
org default - worth checking when a repo's PR form looks "wrong."
