# Bumblebee Supply-Chain Scan (`reusable-bumblebee-scan.yml`)

- Runs on `pull_request` to the default branch, on a daily schedule
  (19:00 UTC), and via `workflow_dispatch`.
- Delegates to `reusable-bumblebee-scan.yml@v1`, which downloads a pinned
  [Bumblebee](https://github.com/perplexityai/bumblebee) release tarball
  and matches installed packages against the curated `threat_intel/`
  exposure catalog shipped in the same release.
- Posts a PR comment on `pull_request`, opens an idempotent tracking
  issue (labelled `bumblebee-finding` and `security`) on non-PR triggers,
  and uploads the NDJSON report as a workflow artifact.

Example minimal usage:

```yaml
name: Bumblebee Scan
on:
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 19 * * *"  # 19:00 UTC = 04:00 JST
  workflow_dispatch:
permissions:
  contents: read
  pull-requests: write
  issues: write
jobs:
  scan:
    uses: geolonia/.github/.github/workflows/reusable-bumblebee-scan.yml@v1
```

## When to use it

Enable on any repo that pulls in third-party packages from npm, PyPI, Go
modules, RubyGems, or Composer. The scan also covers MCP server
configurations and editor / browser extensions in the repo checkout.

## Inputs

| Input | Default | Purpose |
| --- | --- | --- |
| `bumblebee-version` | `v0.1.1` | Upstream release tag. Also the source for the `threat_intel/` catalog. |
| `fail-on-findings` | `true` | Set to `false` for warn-only mode while triaging. |
| `ecosystems` | (all) | Comma-separated list to scope the scan, for example `npm,mcp`. |
| `profile` | `project` | One of `baseline`, `project`, or `deep`. |

## Outputs on a finding

- **`pull_request`**: PR comment (idempotent via marker), inline
  annotations on the matched source files, NDJSON artifact, failed check.
- **Any non-PR trigger** (the default template enables `schedule` and
  `workflow_dispatch`; callers may also add `release` or `push`): opens
  or updates a tracking issue in the same repo with the
  `bumblebee-finding` and `security` labels. The issue body links back to
  the workflow run; re-detections add a comment so the issue bumps on
  notifications.
- Falls back gracefully on fork PRs and Dependabot PRs where
  `GITHUB_TOKEN` is read-only. Inline annotations and the artifact still
  carry the report.

## Adding it to a repo

Open the repo on GitHub and go to **Actions → New workflow**. In the
picker, look under **By Geolonia** for **Bumblebee Supply-Chain Scan**
and click **Configure**. Commit the file as suggested. No other setup
is required.
