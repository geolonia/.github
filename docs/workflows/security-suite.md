# Security Suite

Geolonia's consolidated per-PR security gate. One workflow fans out to four
scanners and posts a single "Security suite" PR comment:

- **bumblebee** - supply-chain package inventory vs. threat-intel catalogs
- **betterleaks** - secret-leak gate on the PR diff
- **pinact** - GitHub Actions SHA-pinning check
- **zizmor** - GitHub Actions static analysis

## How it is enforced

The suite runs on every targeted repo with **no workflow file in that repo**,
via an organization **ruleset** ("Require workflows to pass before merging")
that points at:

```text
geolonia/.github/.github/workflows/security-suite.yml@refs/tags/v1
```

`security-suite.yml` is a thin caller of `reusable-security-suite.yml`, which
calls the scanner reusables, all pinned to `@v1` so a suite change ships with
no per-caller bump. The same suite is also offered in the "New workflow" picker
(`workflow-templates/security-suite.yml`) for repos that opt in manually.
**Do not enable both on one repo.**

## ⚠️ The one rule you must not forget: keep `v1` a LIGHTWEIGHT tag

**The `v1` tag must be a lightweight tag (a direct ref to a commit), never an
annotated tag object.**

The ruleset resolves the required workflow at `security-suite.yml@refs/tags/v1`.
GitHub's "require workflows" **injector cannot resolve an annotated `v1` tag**:
when `v1` is annotated it **silently produces no run at all** - no check-run,
not even a failure - and the required "Security Suite" check hangs at
**"Expected / waiting for workflow to run" forever**, blocking merge on every
consumer PR. This is not documented by GitHub; it was learned the hard way (the
v1.27-v1.31 outage). Note the failure is silent - `@v1` moving to a new release
is **not** proof the gate works.

Check the tag type any time:

```bash
# Annotated if the two SHAs differ; lightweight if the peel is empty/equal.
git ls-remote https://github.com/geolonia/.github refs/tags/v1 'refs/tags/v1^{}'
```

Floating **reusable** refs (`reusable-security-suite.yml@v1`, the scanner
reusables `@v1`) are **fine** - they are not the tag the ruleset resolves, and
they resolve normally at runtime. Do not SHA-pin them thinking it fixes an
"Expected" hang; that was a red herring during the outage.

## What makes `v1` annotated (and how we prevent it)

`v1` is moved by the release workflow (`reusable-release-auto-on-tag.yml`) on
each `vX.Y.Z` push. If the release tag is **annotated** (`git tag -a`), a naive
`git tag -f v1 <tag>` copies the annotated object into `v1`. Two guards:

1. **The release workflow peels to the commit** (`git tag -f v1 "$TAG^{commit}"`),
   so the moved `v1`/minor tags are always lightweight regardless of the release
   tag's type. This is the durable guard.
2. **Cut release tags lightweight anyway** (`git tag vX.Y.Z`, not `git tag -a`),
   as a backstop.

## Releasing a change to the suite

Because everything floats on `@v1`, a suite or scanner change is a **single
release**, no chain-bump:

1. Edit the reusable (`reusable-security-suite.yml` or a scanner reusable),
   open a PR, merge.
2. Cut a `vX.Y.Z` tag on `main` (lightweight). The release workflow moves `v1`
   (lightweight, per the peel above); every consumer on `@v1` picks it up.

## Verify after every release

`@v1` moving is **not** proof the gate works - the injector failure is silent.
After releasing, confirm the suite actually runs on a real consumer PR:

```bash
# Re-trigger an open consumer PR with an empty commit (synchronize event):
gh api "repos/geolonia/<repo>/actions/runs?head_sha=<head>" \
  --jq '.workflow_runs[].name'
# Expect a "Security Suite" run, not just the repo's own CI.
```

If you see only the repo's own CI and no "Security Suite" run, **check the `v1`
tag type first** - it is almost always an annotated `v1`.

## geolonia/.github is not a ruleset target

The repo that **hosts** the required workflow must **not** also be in the
ruleset's target repos. On its own PRs GitHub runs the local branch copy of
`security-suite.yml` (the normal job checks), but the injected required check
expects the `@v1` version and hangs at "Expected". Keep `geolonia/.github`
excluded; it dogfoods the suite via its own `.github/workflows/security-suite.yml`.

## Troubleshooting

See the shared tips in [Reusable Workflow Templates](../workflows.md#troubleshooting).
The failure mode unique to this suite is the silent "Expected" hang above;
it is almost always an **annotated `v1` tag**.
