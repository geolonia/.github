# Reusable GitHub Actions workflows

This repo hosts org-wide reusable workflows. The pattern uses **two
locations** in this repo plus a **tag-pinning convention** in callers.

## The three roles

| Role | Path | Purpose |
|---|---|---|
| **Template** | `workflow-templates/<name>.yml` + `<name>.properties.json` | Shown in GitHub's "New workflow" picker. Points at a tagged reusable workflow. |
| **Entrypoint** | `.github/workflows/<name>.yml` | This repo's own use of the workflow (push/tag triggers). Forwards to the reusable workflow with defaults. |
| **Reusable** | `.github/workflows/reusable-<name>.yml` | The actual logic. Callable from any repo via `uses: geolonia/.github/.github/workflows/reusable-<name>.yml@<tag>`. |

## Currently maintained workflows

- **`publish-techdocs`** - TechDocs publish on `docs/**` or `mkdocs.yml`
  changes. Uses OIDC for AWS, with `TECHDOCS_AWS_ACCOUNT_ID` org secret
  by default; per-repo `AWS_ACCOUNT_ID` override supported.
  **Naming asymmetry**: the template + entrypoint pair are
  `publish-techdocs.yml`, but the reusable file is
  `reusable-backstage-techdocs.yml`, not `reusable-publish-techdocs.yml`
  as the convention in the table above would predict. Consumers `uses:`
  the reusable filename, so any reference must spell out
  `reusable-backstage-techdocs.yml`.
- **`release-auto-on-tag`** - semver-tag-triggered releases (`v*.*.*`).
  Optional inputs control prereleases, release notes, custom regex.
- **`cdk-deploy-monitor`** - runs alongside a CDK deploy job, polls
  CloudFormation/CloudWatch/ECS, asks GitHub Models (GPT-4o) whether the
  deploy is hung, optionally cancels on CANCEL verdict. Requires
  `CdkDeployMonitor` IAM permission bundle and `models: read` permission
  in the calling workflow + every parent in the call chain.
- **`sync-team-access`** - keeps GitHub team-to-repo access in sync.
- **`security-suite`** - consolidated per-PR security gate (bumblebee,
  betterleaks, pinact, zizmor) posting ONE PR comment. Enforced org-wide as a
  ruleset "required workflow" at `security-suite.yml@v1`, so targeted repos
  need NO file of their own. `security-suite.yml` -> `reusable-security-suite.yml`
  -> the scanner reusables, all floating on `@v1`. **CRITICAL: the `v1` tag
  must be LIGHTWEIGHT** (see the warning below). Full page:
  `docs/workflows/security-suite.md`.

## Caller-side conventions

Consumers should pin to a tag, not a SHA, not `@main`:

```yaml
jobs:
  publish:
    uses: geolonia/.github/.github/workflows/reusable-backstage-techdocs.yml@v1
    secrets: inherit
```

## Updating a reusable workflow

When making **non-breaking** changes:

1. Edit `.github/workflows/reusable-<name>.yml`.
2. Open a PR. Wait for CodeRabbit + human approval.
3. Merge to `main`.
4. Move the `v1` tag forward (or whatever major tag is active):

   ```sh
   git tag -f v1
   git push -f origin v1
   ```

When making **breaking** changes:

1. Cut a new major tag (`v2`, `v3`, …) so consumers upgrade on their own
   schedule.
2. Update `workflow-templates/<name>.yml` to reference the new tag - but
   only after enough consumers have migrated, otherwise new "New workflow"
   selections will land on the latest tag and consumers expecting `v1`
   stay on `v1`.
3. Document the breaking change in a release note on the new tag.

Always keep `workflow-templates/<name>.properties.json` in sync with the
YAML so the picker shows accurate names/descriptions.

## CRITICAL: the Security Suite `v1` tag must stay LIGHTWEIGHT

`security-suite.yml` is a ruleset **required workflow** resolved at
`@refs/tags/v1`. GitHub's "require workflows" injector **cannot resolve an
ANNOTATED `v1` tag** - it silently produces NO run, and the required
"Security Suite" check hangs at "Expected / waiting for workflow to run"
forever on every consumer PR. (This caused an org-wide outage: v1.27-v1.31,
2026-07. The reusable-ref float-vs-SHA was a RED HERRING; the real cause was
an annotated `v1`.)

- Cut release tags **lightweight** (`git tag vX.Y.Z`, NOT `git tag -a`). An
  annotated `vX.Y.Z` gets copied into `v1` by the release workflow's
  `git tag -f v1 <tag>`.
- `reusable-release-auto-on-tag.yml` now peels to the commit
  (`git tag -f v1 "$TAG^{commit}"`) so moved tags are lightweight regardless -
  keep that.
- Floating reusable refs (`reusable-security-suite@v1`, scanners `@v1`) are
  FINE; do NOT SHA-pin them to "fix" an Expected hang - check the `v1` tag
  type instead:
  `git ls-remote https://github.com/geolonia/.github refs/tags/v1 'refs/tags/v1^{}'`
  (annotated if the two SHAs differ).
- After any suite release, VERIFY on a real consumer PR that a "Security Suite"
  run appears (`@v1` moving is not proof - the failure is silent).
- `geolonia/.github` must NOT be in the ruleset's target repos (it hosts the
  required workflow; its own PRs would hang the same way).

## Common troubleshooting

- **`Could not find reusable workflow`** - caller is pinned to a tag that
  doesn't exist (or was deleted). Check
  `gh api repos/geolonia/.github/git/refs/tags --jq '.[].ref'`.
- **`Input required and not supplied`** for AWS account ID - confirm the
  org-level `TECHDOCS_AWS_ACCOUNT_ID` secret is visible to the calling
  repo, or the per-repo `AWS_ACCOUNT_ID` override is set.
- **CDK deploy monitor doesn't post comments** - needs `contents: write`
  + `models: read` on the **caller's** top-level `permissions:` block.
- **Security Suite required check stuck at "Expected"** on consumer PRs with no
  "Security Suite" run in Actions - the `v1` tag is ANNOTATED. Recreate it
  lightweight (`git tag -d v1; git tag v1 <commit>; git push -f origin v1`).
  See the CRITICAL section above.
