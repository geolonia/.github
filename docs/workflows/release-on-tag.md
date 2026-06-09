# Release on Tag (`release-auto-on-tag.yml`)

- Triggers on semver-like tags (`v*.*.*`).
- Delegates to `reusable-release-auto-on-tag.yml@v1` to create releases and move
  major/minor tags if configured.
- Inherits calling-repo secrets; optional inputs allow toggling prereleases,
  generating release notes, or customizing the tag regex.

Example minimal usage:

```yaml
jobs:
  publish:
    uses: geolonia/.github/.github/workflows/reusable-release-auto-on-tag.yml@v1
    secrets: inherit
```

See the general [Troubleshooting](../workflows.md#troubleshooting) section for
log-fetching and dispatch tips.
