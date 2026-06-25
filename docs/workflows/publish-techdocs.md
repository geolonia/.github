# Publish TechDocs (`publish-techdocs.yml`)

- Runs on `main` when `docs/**` or `mkdocs.yml` changes, or manually via `workflow_dispatch`.
- Delegates to `reusable-backstage-techdocs.yml@v1` with safe defaults for AWS
  region, environment, and tool versions.
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
