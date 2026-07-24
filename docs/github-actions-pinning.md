# Pinning GitHub Actions

Our org standard is to pin every GitHub Action to a full 40-character commit
SHA with a readable version comment, and let automation keep it current:

```yaml
uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd  # v6.0.2
```

This page explains why and how to adopt it. The reusable CI check itself is
documented in the
[Action Pinning Check](workflows/pinact-check.md) page.

## Why pin to a commit SHA

A tag like `@v4` is **mutable**: whoever controls the action's repository can
move it to point at different code at any time, even after you have reviewed it.
Pinning to a tag trusts that it will never be changed or hijacked.

A 40-character commit SHA is **immutable**: it always resolves to the exact
commit you reviewed. So a moved or hijacked tag cannot silently swap in
different code on your next workflow run, where it would have access to your
`GITHUB_TOKEN` and secrets. That is what pinning to a SHA prevents. The trailing
`# v6.0.2` comment keeps the line readable. The classic downside of SHA pinning
is staleness, which we remove with Dependabot.

## How the pieces fit together

Three tools enforce one rule: **nothing younger than 7 days**.

| Tool | Role |
| --- | --- |
| **pinact** | Pins actions (writes the SHA + version comment) and verifies them. The Action Pinning Check CI gate runs pinact in validation-only mode and fails a PR if any action is unpinned, has a mismatched comment, or is younger than the minimum release age. It never edits files in CI. |
| **Dependabot** | Maintains pins. Weekly it opens PRs that bump the SHA and the comment together, deferred by a `cooldown`. |
| **`.pinact.yml`** | Sets the `min_age` (7 days) and exempts our own `geolonia/*` actions from the wait. |

The 7-day wait is the same idea as pnpm's `minimumReleaseAge`, which we also use
for npm dependencies: do not adopt a release until it has survived a week, the
window in which a malicious release is most likely to be caught and pulled. The
Dependabot `cooldown` is set to **8** days (one more than pinact's 7) because
Dependabot counts whole calendar days while pinact enforces an exact 168-hour
floor, so the extra day guarantees Dependabot PRs pass the check on arrival.

## Adopting it in a repo

New repos created from the Backstage scaffolder get everything below out of the
box. For an existing repo:

1. Copy the canonical files from this repository:
   - the repo-root
     [`.pinact.yml`](https://github.com/geolonia/.github/blob/main/.pinact.yml)
     to your repo root as `.pinact.yml`. This is optional: the Action Pinning
     Check fetches this same file as its fallback when your repo has none, so
     copy it only if you want to pin a local override.
   - [`pinact/dependabot.yml`](https://github.com/geolonia/.github/blob/main/pinact/dependabot.yml)
     to `.github/dependabot.yml` (merge the `github-actions` entry if you
     already have one). This one is required: Dependabot only reads it in-repo.
   - optional: `pinact/.pre-commit-config.example.yaml` to your repo root as
     `.pre-commit-config.yaml` (auto-pins on every commit)
2. Add the **Action Pinning Check** workflow, either way:
   - **From the GitHub UI:** in your repo, open **Actions -> New workflow**,
     and under **By Geolonia** choose **Action Pinning Check**, then
     **Configure** and commit the suggested file.
   - **By copying the file:** see the
     [Action Pinning Check](workflows/pinact-check.md) reference.
3. Run `pinact run` once (or let the pre-commit hook do it) to pin the existing
   workflows, then commit.

## Day to day

- **Never hand-edit a SHA.** Write the normal tag (`actions/setup-node@v6`) and
  let pinact pin it, via the pre-commit hook or `pinact run`. If the Action
  Pinning Check ever fails, the fix is to run `pinact run`, never to edit the
  SHA by hand.
- **Dependabot does the bumping.** It opens weekly PRs that update the SHA and
  the version comment together: minor and patch grouped into one PR, majors
  individually. Skim the linked changelog, confirm CI is green, and merge.

## See also

- [Action Pinning Check reusable workflow](workflows/pinact-check.md)
- Canonical config: the repo-root
  [`.pinact.yml`](https://github.com/geolonia/.github/blob/main/.pinact.yml),
  fetched as the Action Pinning Check fallback. Per-repo helpers live in
  [`pinact/`](https://github.com/geolonia/.github/tree/main/pinact)
  (`dependabot.yml`, `.pre-commit-config.example.yaml`).
- [pinact](https://github.com/suzuki-shunsuke/pinact) upstream documentation.
