# Security Suite

Geolonia's consolidated per-PR security gate. One workflow fans out to four
scanners and posts a single "Security suite" PR comment:

- **bumblebee** - supply-chain package inventory vs. threat-intel catalogs
- **betterleaks** - secret-leak gate on the PR diff
- **pinact** - GitHub Actions SHA-pinning check (warn-only)
- **zizmor** - GitHub Actions static analysis (warn-only)

## How it is enforced

The suite runs on every targeted repo with **no workflow file in that repo**,
via an organization **ruleset** ("Require workflows to pass before merging")
that points at:

```
geolonia/.github/.github/workflows/security-suite.yml@refs/tags/v1
```

`security-suite.yml` is a thin caller of `reusable-security-suite.yml`, which
calls the scanner reusables. The same suite is also offered in the "New
workflow" picker (`workflow-templates/security-suite.yml`) for repos that opt
in manually instead of via the ruleset. **Do not enable both on one repo.**

## ⚠️ The one rule you must not forget on a release

**The required-workflow reusable tree must be pinned to immutable SHAs at every
level. A moving tag (`@v1`) anywhere in the tree silently breaks the gate.**

```
security-suite.yml              (required workflow, ruleset points here @v1)
  └─ uses: reusable-security-suite.yml@<SHA>        # MUST be a SHA
       ├─ uses: reusable-bumblebee-scan.yml@<SHA>   # MUST be a SHA
       ├─ uses: reusable-secret-leak-check.yml@<SHA> # MUST be a SHA
       └─ uses: reusable-pinact-check.yml@<SHA>     # MUST be a SHA
```

Why: the ruleset "require workflows" **injector** runs a required workflow only
when its **entire** reusable tree resolves to immutable refs. If any ref in the
tree is a moving tag, the injector **silently produces no run at all** - no
check-run, not even a failure - and the required "Security Suite" check hangs at
**"Expected / waiting for workflow to run" forever**, blocking merge on every
consumer PR. This is not documented clearly by GitHub; it was learned the hard
way (the v1.27-v1.29 regression, see
[#88](https://github.com/geolonia/.github/pull/88) /
[#89](https://github.com/geolonia/.github/pull/89) /
[#90](https://github.com/geolonia/.github/pull/90)).

This rule applies **only to the required-workflow tree above.** The picker
template `workflow-templates/security-suite.yml` may keep `@v1` - it is adopted
as a normal, non-injected workflow, where moving tags resolve fine at runtime.

## Releasing a change to the suite (the two-release dance)

Because `security-suite.yml` pins `reusable-security-suite.yml` by SHA, and that
file pins the scanners by SHA, a change has to converge over **two releases**
(the SHAs form a caller → callee chain and cannot both point at the same
not-yet-created commit):

1. **Change the inner piece + pin.** Edit the scanner reusable (or
   `reusable-security-suite.yml`), and in `reusable-security-suite.yml` pin the
   scanner refs to the SHA of the latest release that contains your change.
   Merge and release (e.g. `vX`).
2. **Bump the top ref.** In `security-suite.yml`, bump the
   `reusable-security-suite.yml` ref to `vX`'s SHA. Merge and release (`vY`).

After `vY`, `@v1` fast-forwards to it and the whole tree is immutable again.

Update the pin comment (`@<sha> # vX.Y.Z`) so pinact stays green.
`.pinact.yml` and `zizmor.yml` exempt `geolonia/*` reusables referenced at a
bare major tag from pinning **for the template's sake** - do not rely on that
exemption for the required-workflow tree; that tree must be SHA-pinned.

## Verify after every release

`@v1` moving is **not** proof the gate works - the injector failure is silent.
After releasing, confirm the suite actually runs on a real consumer PR:

```bash
# Re-trigger an open consumer PR with an empty commit (synchronize event),
# then confirm a "Security Suite" run appears on the new head SHA:
gh api "repos/geolonia/<repo>/actions/runs?head_sha=<head>" \
  --jq '.workflow_runs[].name'
```

If you see only the repo's own CI and no "Security Suite" run, the tree still
has a moving ref somewhere - re-check every `uses:` in the chain above.

## geolonia/.github is not a ruleset target

The repo that **hosts** the required workflow must **not** also be in the
ruleset's target repos. On its own PRs GitHub runs the local branch copy of
`security-suite.yml` (the normal job checks), but the injected required check
expects the `@v1` version and hangs at "Expected". Keep `geolonia/.github`
excluded from the ruleset targets; it dogfoods the suite via its own
`.github/workflows/security-suite.yml` instead.

## Troubleshooting

See the shared tips in [Reusable Workflow Templates](../workflows.md#troubleshooting).
The failure mode unique to this suite is the silent "Expected" hang described
above; it is almost always a moving ref left somewhere in the required-workflow
tree.
