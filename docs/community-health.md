# Community Health Files

The `.github` repository applies these policies and templates to all Geolonia
repositories automatically:

- `CODE_OF_CONDUCT.md`: expected behavior and reporting guidance (based on
  Contributor Covenant)
- `CONTRIBUTING.md`: contribution flow, code style, and review expectations
- `SUPPORT.md`: how to request help or report issues
- `SECURITY.md`: security contacts and vulnerability reporting process
- Issue templates in `.github/ISSUE_TEMPLATE/`:
  - `bug_report.yml`: gathers what happened, expectations, steps to reproduce,
    and logs; labels issues with `bug`.
  - `feature_request.yml`: captures desired outcome, motivation, alternatives,
    and extra context; labels issues with `enhancement`.
  - `config.yml`: allows blank issues and points to commercial support.
- `.github/pull_request_template.md`: prompts for linked issue (`Fixes #...`),
  summary, and optional tests/docs checklist

## Maintenance tips

- Keep contact emails/links current in the policy files.
- When guidelines change, update this repo first so every project inherits the change.
- Avoid repository-specific information here; use project-level overrides if needed.
- Ensure no sensitive information is included, as this repo must remain public.
