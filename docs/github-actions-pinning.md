# Pinning GitHub Actions

Our org standard is to pin every GitHub Action to a full 40-character commit
SHA with a readable version comment, and let automation keep it current:

```yaml
uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd  # v6.0.2
```

This page is the "why and how" for developers. For the reusable CI check and
the exact files to copy, see the
[Action Pinning Check](workflows.md#action-pinning-check-reusable-pinact-checkyml)
section of the Reusable workflows page.

## Why pin to a commit SHA

A tag like `@v4` is **mutable**: whoever controls the action's repository can
move it to point at different code at any time. If a popular action is
compromised (this is not hypothetical, it happened to `tj-actions/changed-files`
in 2026), every workflow that uses the moved tag silently runs the attacker's
code on its next run, with access to that workflow's `GITHUB_TOKEN` and secrets.

A 40-character commit SHA is **immutable**: it always resolves to the exact
commit you reviewed, so a hijacked tag cannot change what runs in your CI. The
trailing `# v6.0.2` comment keeps the line readable so you can still tell at a
glance which version is pinned.

The classic downside of SHA pinning is staleness (pinned SHAs do not pick up
fixes on their own). We remove that downside with Dependabot, which bumps the
SHA and rewrites the version comment for us.

## How the pieces fit together

Three tools enforce one simple rule: **nothing younger than 7 days**.

| Tool | Role |
| --- | --- |
| **pinact** | Pins actions (writes the SHA + version comment) and verifies them. The Action Pinning Check CI gate runs pinact in validation-only mode and fails a PR if any action is unpinned, has a mismatched comment, or is younger than the minimum release age. It never edits files in CI. |
| **Dependabot** | Maintains pins. Weekly it opens PRs that bump the SHA and the comment together. A `cooldown` defers brand-new releases. |
| **`.pinact.yml`** | Sets the `min_age` (7 days) used by pinact, and exempts our own `geolonia/*` actions from the wait. |

The 7-day wait is the same idea as pnpm's `minimumReleaseAge`, which we also use
for npm dependencies: do not adopt a release until it has survived a week, the
window in which a hijacked or malicious release is most likely to be caught and
pulled. One consistent rule across both npm packages and GitHub Actions.

Two numbers that look almost the same and why they differ:

- pinact `min_age` is **7** days (an exact 168 hour floor).
- Dependabot `cooldown` is **8** days. Dependabot counts whole calendar days,
  so setting it one day higher guarantees its PRs are always past pinact's
  exact 168 hour floor when they open, and therefore pass the CI check on
  arrival instead of failing it for a few hours at the day boundary.

## Adding or changing an action

**Do not hand-write or hand-edit the SHA.** Write the normal human-readable tag
and let the tooling pin it:

```yaml
# write this:
uses: actions/setup-node@v6
# pinact turns it into:
uses: actions/setup-node@48b55a011bda9f5d6aeb4c2d9c7362e8dae4041e  # v6.4.0
```

Easiest path, install the pre-commit hook once per laptop so commits are pinned
automatically:

```bash
brew install pinact pre-commit          # pinact tap: suzuki-shunsuke/pinact
cp /path/to/geolonia/.github/pinact/.pre-commit-config.example.yaml .pre-commit-config.yaml
pre-commit install
```

After that, `git commit` runs pinact on changed workflow files and pins anything
new. You can also run it manually any time with `pinact run`.

To intentionally move an action to a newer version, run `pinact run --update`
(it still honors the 7-day minimum age). In day-to-day work you rarely need to:
Dependabot does the bumping for you.

## Handling a Dependabot pull request

Dependabot opens these weekly. Each one has already applied the bump (SHA +
version comment) and links the upstream changelog.

- **Minor and patch** bumps arrive grouped into a single PR. Skim the changelog,
  confirm CI is green (including the Action Pinning Check), and merge.
- **Major** bumps arrive as individual PRs so you can review the breaking
  changes on their own.

You never edit the SHA yourself; just review and merge.

## When the Action Pinning Check fails

The check is failing for one of these reasons. The fix is almost always "let
pinact do it," never "hand-edit the SHA."

| Message | Cause | Fix |
| --- | --- | --- |
| GitHub Actions aren't pinned | An action still uses a tag or branch | Run `pinact run` (or let the pre-commit hook run), commit the result |
| Version comment does not match the SHA | The `# vX.Y.Z` comment drifted from the pinned commit | Run `pinact run` to rewrite the comment |
| Action version is younger than the min-age cutoff | The pinned release is less than 7 days old | Wait until it ages past 7 days, or pin the previous (older) release. This is the cooldown protecting you from brand-new releases |

## FAQ

**Does SHA pinning mean we miss security updates?**
No. Dependabot bumps pins weekly (the SHA and the comment together), so fixes
still flow in through reviewed PRs.

**Why keep the version comment if the SHA is what matters?**
Readability. pinact keeps the comment in sync with the SHA, so it is safe to
trust at a glance and never drifts.

**Do I pin first-party actions like `actions/checkout` too?**
Yes, all actions are pinned, first-party included. Our own `geolonia/*` reusable
workflows are pinned as well; Dependabot maintains them, and they are exempt
only from the 7-day cooldown (we author and control those releases, so the
hijacked-third-party-release risk the cooldown guards against does not apply).

**Do I need `persist-credentials: false` on `actions/checkout`?**
Not on v6 or newer. As of checkout v6 the token is stored outside the repo
working tree, which closes the path where it could leak through an uploaded
artifact. Setting `persist-credentials: false` is optional hygiene on v6+, not a
requirement. It still matters on v5 and earlier.

**How do I adopt the standard in a new or existing repo?**
Copy this repo's canonical `pinact/.pinact.yml` to your repo root as
`.pinact.yml`, copy `.github/dependabot.yml`, add the Action
Pinning Check workflow (see the
[Reusable workflows](workflows.md#action-pinning-check-reusable-pinact-checkyml)
page), then run `pinact run` once to pin the existing workflows. New repos
created from the Backstage scaffolder get all of this out of the box.

## See also

- [Action Pinning Check reusable workflow](workflows.md#action-pinning-check-reusable-pinact-checkyml)
- Canonical config to copy: `pinact/.pinact.yml` and
  `pinact/.pre-commit-config.example.yaml` in this repository.
- [pinact](https://github.com/suzuki-shunsuke/pinact) and
  [Dependabot cooldown](https://docs.github.com/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file#cooldown)
  upstream docs.
