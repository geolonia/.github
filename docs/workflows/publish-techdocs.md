# Publish TechDocs (`publish-techdocs.yml`)

- Runs on `main` when `docs/**` or `mkdocs.yml` changes, or manually via `workflow_dispatch`.
- Delegates to `reusable-backstage-techdocs.yml@v1` with safe defaults for AWS
  region, environment, and tool versions.
- Uses `TECHDOCS_AWS_ACCOUNT_ID` of `geolonia/.github` repository secret by default;
  optional inputs allow setting a repo secret `AWS_ACCOUNT_ID` to override on a
  per-repo basis.

Example minimal usage after selecting the template:

```yaml
jobs:
  publish:
    uses: geolonia/.github/.github/workflows/reusable-backstage-techdocs.yml@v1
    # Optional repo override:
    # with:
    #   environment: production
    #   aws_region: ap-northeast-1
    secrets: inherit
    # In case you want to override the AWS account ID:
    # secrets: 
    #   AWS_ACCOUNT_ID: ${{ secrets.TECHDOCS_AWS_ACCOUNT_ID }}
```

## Troubleshooting

### TechDocs publish fails with AWS credentials error

Ensure the workflow environment has access to the OIDC-based credentials.
Check that the AWS account ID secret (`TECHDOCS_AWS_ACCOUNT_ID` or the repo
override `AWS_ACCOUNT_ID`) is set and non-empty, and that the IAM role trust
policy allows the `geolonia` GitHub org.

See also the general [Troubleshooting](../workflows.md#troubleshooting) section
for log-fetching and dispatch tips.
