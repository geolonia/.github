# Publish TechDocs (`publish-techdocs.yml`)

- Runs on `main` when `docs/**` or `mkdocs.yml` changes, or manually via `workflow_dispatch`.
- A manual run on any ref other than the publish branch is skipped: the
  `branches:` filter does not apply to `workflow_dispatch`, so the job checks
  the ref itself.
- The reusable workflow refuses to publish a commit that is not on the publish
  branch (the tip or an ancestor of it), whatever the caller passes in `tag`.
  Release tags must point at merged commits (with squash merges, the
  squash-merge commit on the publish branch, not the PR branch commit).
- The publish branch is the default branch. A repository whose default branch
  is not the one it publishes docs from (for example `develop` as default and
  docs published on pushes to `main`) sets the repository variable
  `TECHDOCS_PUBLISH_BRANCH` to that branch (Settings, Secrets and variables,
  Actions, Variables). Without it, every publish from `main` fails with
  "not on the publish branch", because `main` is ahead of `develop`. It is a
  variable and not an input, because only repository admins can change
  variables. Set it on the repository, not on the organization (it would apply
  to every repository) and not on an environment (the template's dispatch
  guard does not see environment variables). The check is only as strong as
  the protection of that branch: it must require pull requests, as the org
  ruleset does for the default branch, `main` and `master`. Also make the
  `push:` trigger of the caller list that branch.
- Delegates to `reusable-backstage-techdocs.yml@v1` with safe defaults for AWS
  region, environment, and tool versions.
- The reusable workflow accepts only plain version strings for the tools it
  installs, because they are installed in the job that holds the publisher's
  AWS role. It stops with an error, before the checkout, if a value (input or
  the matching variable) does not match:

  | Input (variable) | Default | Accepted |
  | --- | --- | --- |
  | `techdocs_cli_version` (`TECHDOCS_CLI_VERSION`) | `latest` | a semver version or range (`1.9.0`, `^1.9.0`, `~1.9`, `1.x`, `>=1.9.0 <2.0.0`) or a dist-tag (`latest`, `next`) |
  | `mkdocs_core_version` (`MKDOCS_TECHDOCS_CORE_VERSION`) | `>=1.6,<2.0` | comma-separated pip version specifiers (`>=1.6,<2.0`, `==1.6.1`, `~=1.6`) |
  | `node_version` (`TECHDOCS_NODE_VERSION`) | `24` | a Node.js version (`24`, `24.1.0`, `24.x`, `lts/*`) |
  | `python_version` (`TECHDOCS_PYTHON_VERSION`) | `3.13` | a Python version (`3.13`, `3.13.1`, `3.x`, `pypy3.10`) |
  | `build_directory` (`TECHDOCS_BUILD_DIR`) | `site` | a relative directory inside the checkout (`site`, `./site`, `build/site/`), no `..` and no leading `/` |

  Git, URL, file and alias specs (`github:owner/repo`, `npm:other@1`,
  `file:...`, ` @ https://...`), pip extras and markers are refused. Docs are
  generated into `build_directory`. The workflow also stops, before
  generation and again before publishing, if any symlink in the checkout
  (a docs page, the build directory, or a link inside a symlinked directory)
  leads outside the checkout.

  These checks are in the reusable workflow, so a caller gets them only from a
  pin that includes them. Callers pinned to `@v1` get them with the release;
  callers pinned to a commit SHA of an earlier version (v1.43.5 or older) must
  update the pin. New repositories get the current pin from the template.
- Uses `TECHDOCS_AWS_ACCOUNT_ID` of `geolonia/.github` repository secret by default;
  optional inputs allow setting a repo secret `AWS_ACCOUNT_ID` to override on a
  per-repo basis.

The template is standalone by default (`push` on `docs/**` plus
`workflow_dispatch`). Pass the AWS account secrets explicitly rather than
`secrets: inherit`, so the reusable receives only what it needs.

Example minimal usage after selecting the template:

```yaml
jobs:
  publish:
    uses: geolonia/.github/.github/workflows/reusable-backstage-techdocs.yml@v1
    # Optional repo override:
    # with:
    #   environment: production
    #   aws_region: ap-northeast-1
    secrets:
      AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
      TECHDOCS_AWS_ACCOUNT_ID: ${{ secrets.TECHDOCS_AWS_ACCOUNT_ID }}
```

To also publish docs as part of a release (called from `publish-release.yml`),
add a `workflow_call` trigger that declares an explicit `secrets` contract
(`AWS_ACCOUNT_ID` and `TECHDOCS_AWS_ACCOUNT_ID`, both `required: false`) and
have the caller pass them explicitly.

## Troubleshooting

### TechDocs publish fails with AWS credentials error

Ensure the workflow environment has access to the OIDC-based credentials.
Check that the AWS account ID secret (`TECHDOCS_AWS_ACCOUNT_ID` or the repo
override `AWS_ACCOUNT_ID`) is set and non-empty, and that the IAM role trust
policy allows the `geolonia` GitHub org.

See also the general [Troubleshooting](../workflows.md#troubleshooting) section
for log-fetching and dispatch tips.
