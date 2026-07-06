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
  betterleaks, pinact, zizmor) posting ONE PR comment. Enforced org-wide as
  a ruleset "required workflow" pointing at `security-suite.yml@v1`, so
  targeted repos need NO file of their own. `security-suite.yml` is a thin
  caller of `reusable-security-suite.yml`, which calls the scanner reusables.
  **SPECIAL RELEASE RULES apply - see the warning below. Do not just move
  `v1`.** Full page: `docs/workflows/security-suite.md`.

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

## SPECIAL CASE: the Security Suite required-workflow tree

The "move `v1` forward" flow above is UNSAFE for `security-suite.yml` and the
reusables it pulls in. Because it is a ruleset **required workflow**, the
"require workflows" injector runs it ONLY when its ENTIRE reusable tree
resolves to immutable SHAs:

```
security-suite.yml  ->  reusable-security-suite.yml@<SHA>
                          ->  reusable-{bumblebee-scan,secret-leak-check,pinact-check}.yml@<SHA>
```

A moving tag (`@v1`) ANYWHERE in that tree makes the injector **silently
produce no run at all** - no check-run, not even a failure - so the required
"Security Suite" check hangs at "Expected / waiting for workflow to run"
FOREVER and blocks merge on every consumer PR. (Learned via the v1.27-v1.29
regression; fixed in #88/#89/#90.) GitHub's docs do not spell this out.

Rules:

- Keep every `uses:` in the tree above SHA-pinned (with a `# vX.Y.Z` comment).
  Only the picker template `workflow-templates/security-suite.yml` may stay
  `@v1` (it is a normal, non-injected workflow).
- Releasing a suite/scanner change takes TWO releases (the SHAs are a
  caller->callee chain): (1) pin the scanner refs inside
  `reusable-security-suite.yml` to the latest release SHA, release `vX`;
  (2) bump `security-suite.yml`'s ref to `vX`'s SHA, release `vY`.
- `@v1` fast-forwarding is NOT proof it works (the failure is silent). After
  release, re-trigger a consumer PR (empty commit) and confirm a
  "Security Suite" run appears on the new head SHA.
- `geolonia/.github` must NOT be in the ruleset's target repos (it hosts the
  required workflow; its own PRs would hang the same way). It dogfoods via its
  local `security-suite.yml` instead.

## Common troubleshooting

- **`Could not find reusable workflow`** - caller is pinned to a tag that
  doesn't exist (or was deleted). Check
  `gh api repos/geolonia/.github/git/refs/tags --jq '.[].ref'`.
- **`Input required and not supplied`** for AWS account ID - confirm the
  org-level `TECHDOCS_AWS_ACCOUNT_ID` secret is visible to the calling
  repo, or the per-repo `AWS_ACCOUNT_ID` override is set.
- **CDK deploy monitor doesn't post comments** - needs `contents: write`
  + `models: read` on the **caller's** top-level `permissions:` block.
- **Security Suite required check stuck at "Expected"** on consumer PRs, with
  no "Security Suite" run in the Actions tab - a moving `@v1` ref is present
  somewhere in the required-workflow tree. SHA-pin every `uses:` in the tree
  (see the SPECIAL CASE section above).
