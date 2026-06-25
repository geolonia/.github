# Release on Tag (`release-auto-on-tag.yml`)

- Triggers on semver-like tags (`v*.*.*`).
- Delegates to `reusable-release-auto-on-tag.yml@v1` to create releases and move
  major/minor tags if configured.
- Needs only `contents: write` (to create the release and move tags) and the
  two `WORKFLOWS_*` GitHub App secrets. Grant the permission on the job and pass
  those secrets explicitly rather than `secrets: inherit`, so the reusable
  receives only what it needs.
- Optional inputs allow toggling prereleases, generating release notes, or
  customizing the tag regex.

Example minimal usage:

```yaml
jobs:
  publish:
    permissions:
      contents: write
    uses: geolonia/.github/.github/workflows/reusable-release-auto-on-tag.yml@v1
    secrets:
      WORKFLOWS_CLIENT_ID: ${{ secrets.WORKFLOWS_CLIENT_ID }}
      WORKFLOWS_APP_PRIVATE_KEY: ${{ secrets.WORKFLOWS_APP_PRIVATE_KEY }}
```

See the general [Troubleshooting](../workflows.md#troubleshooting) section for
log-fetching and dispatch tips.
