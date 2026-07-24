# Action Pinning Check (`reusable-pinact-check.yml`)

- Runs on `pull_request` to the default branch.
- Delegates to `reusable-pinact-check.yml@v1`, which runs
  [pinact](https://github.com/suzuki-shunsuke/pinact) (via
  `suzuki-shunsuke/pinact-action`) in validation-only mode (`fix: false`):
  it never edits a file or pushes a commit, it only fails the check.
- Fails the PR when any GitHub Action or reusable workflow is not pinned
  to a full commit SHA, when a version comment does not match the pinned
  SHA, or when a pinned release is younger than the minimum release age.

The org standard is to pin every action to a 40-char commit SHA with a
matching version comment, for example
`actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd  # v6.0.2`.
SHA pins are immutable, so a hijacked upstream tag cannot silently change
what runs in CI. Dependabot keeps the pins current.

Example minimal usage:

```yaml
name: Action Pinning Check
on:
  pull_request:
    branches: [main]
permissions:
  contents: read
jobs:
  pinact:
    uses: geolonia/.github/.github/workflows/reusable-pinact-check.yml@v1
```

## Configuration files

One file is required in the repo; the other is optional:

1. **`.github/dependabot.yml`** (required, copy `pinact/dependabot.yml`) with a
   `github-actions` ecosystem entry,
   `cooldown: { default-days: 8 }`, and a `groups` block batching
   minor/patch bumps into one PR while majors arrive individually.
   Dependabot bumps the SHA and the version comment together, so pins
   never go stale. The cooldown is one day longer than the pinact
   `min_age` of 7: Dependabot counts whole calendar days while pinact
   enforces an exact 168h floor, so an 8-day cooldown guarantees a
   Dependabot PR always clears the Action Pinning Check on arrival
   instead of failing it for a few hours at the day boundary. Dependabot
   only reads this file in-repo, so it cannot come from the check's fallback.
2. **`.pinact.yml`** (optional, copy the repo-root
   [`.pinact.yml`](https://github.com/geolonia/.github/blob/main/.pinact.yml)):
   sets the minimum release age (org default: 7 days, `min_age.value: 7`,
   `always: true`). This check fetches the same canonical file as its fallback
   when your repo has none, so commit a local copy only to pin an override.
   It is the GitHub Actions analog of pnpm's `minimumReleaseAge`: a new
   tag is not adopted until it has survived a week, the window when a
   hijacked-tag attack is most likely to still be live. The canonical
   file exempts `geolonia/*` from the cooldown (we author our own
   releases; the window guards against third-party tag hijacks) and is
   still SHA-pinned.

## Check inputs

| Input | Default | Purpose |
| --- | --- | --- |
| `runs-on` | `ubuntu-latest` | Runner label for the check job. |

## Local pre-commit (optional but recommended)

Install the [pre-commit](https://pre-commit.com/) hook so pins are
applied and the min-age is enforced before you push, instead of finding
out from a red CI check. Copy `pinact/.pre-commit-config.example.yaml`
from `geolonia/.github` to `.pre-commit-config.yaml`, install pinact
(`brew install pinact`), then `pre-commit install`.

## Enabling the check

1. Add `.github/dependabot.yml` (required); optionally add `.pinact.yml` (see
   "Configuration files" above).
2. Open the repo on GitHub and go to **Actions -> New workflow**. Under
   **By Geolonia**, pick **Action Pinning Check** and **Configure**.
3. Run `pinact run` once locally (or let the pre-commit hook do it) to
   SHA-pin existing workflows, then commit.
