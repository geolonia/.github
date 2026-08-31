# Shared CodeRabbit Configuration

CodeRabbit reviews pull requests across Geolonia repositories. Every repository
draws its review settings from a single file in this repo:
[`.coderabbit.yaml`](https://github.com/geolonia/.github/blob/main/.coderabbit.yaml).

## How a repository picks it up

This is **not** GitHub's community health fallback. That mechanism only covers
files like `CODE_OF_CONDUCT.md` and the issue templates, and CodeRabbit does not
take part in it. Placing `.coderabbit.yaml` in this repo has no effect on another
repository by itself.

Sharing works through a CodeRabbit feature called `remote_config`. Each repository
keeps a real `.coderabbit.yaml` at its own root, containing only a pointer:

```yaml
remote_config:
  repository: geolonia/.github
  ref: main
  path: .coderabbit.yaml
```

At review time CodeRabbit reads the repository's own file, sees `remote_config`,
fetches the referenced file, and uses it as the configuration.

To enrol a new repository, add those four lines to its root. That is the whole
setup.

There is also a URL form, pointing at the `raw.githubusercontent.com` address of
the same file. It works, and every Geolonia repository used it until recently, but
CodeRabbit documents it as not recommended, so prefer the repository form above.

## Four consequences worth knowing

**Reviews use the configuration on the base branch, not the head branch.** This is
the one that surprises people. A pull request that adds or edits `.coderabbit.yaml`
is still reviewed with the configuration that was already on the base branch, so a
configuration change never affects its own pull request. Enrolling a repository
takes effect on the next pull request opened after the enrolling one merges. To see
which configuration a review actually used, comment `@coderabbitai configuration`
on the pull request.

**The pointer is resolved at review time, against `main`.** Merging a change to
the shared file changes review behavior everywhere on the next CodeRabbit review,
including the next review of a pull request that is already open. No
per-repository bump is needed, and there is no staged rollout. Revert the change
to roll back.

**By default it is a whole file replacement, not a merge.** `inheritance` defaults
to `false`, which means CodeRabbit stops at the first configuration source it
finds. Any key sitting next to `remote_config` in a repository's local file is
ignored without warning, so pasting a configuration snippet into a repository root
does not add a setting on top of the shared config, it detaches that repository
from the shared config entirely and silently. Setting `inheritance: true` changes
this and merges values from parent levels instead, which is the supported way for a
repository to extend the shared config rather than replace it. Either way, do it
deliberately, not by accident.

**The shared configuration is readable by anyone.** This repository is public, so
the shared file must never contain anything sensitive. It holds review tool toggles
only.

## Editing the shared file

Treat an edit here as a change to every repository's review behavior, because that
is what it is.

- Open a pull request. Do not push to `main`.
- Validate keys against CodeRabbit's schema before pushing. Unknown keys are
  rejected and CodeRabbit posts a configuration error instead of a review:

  ```bash
  curl -sSL https://coderabbit.ai/integrations/schema.v2.json -o /tmp/cr.json
  ```

- Note that `remote_config` itself is **not** in that schema, so the pointer form
  cannot be schema checked. A wrong key there does not raise an error, it falls
  back to defaults silently, which looks exactly like a working configuration until
  someone notices pull requests are no longer being approved. Verify a pointer
  change by commenting `@coderabbitai configuration` on a pull request and reading
  the reported source path, for example:

  ```text
  Configuration used: Path: geolonia/.github/.coderabbit.yaml@main (via .coderabbit.yaml)
  ```

- Keep the file free of settings that only make sense in one repository.
- Prefer small changes, and revert quickly if reviews start behaving oddly.
- Remember that the change will not affect the pull request making it, since
  reviews use the base branch configuration.

## Reaching an approving review

A repository's branch rules require an approving review before merge. CodeRabbit
supplies that approval once every comment it raised is resolved and it has
reviewed the latest commit. Two settings make this work, and both live in the
shared file: `reviews.request_changes_workflow` and
`reviews.auto_review.auto_incremental_review`.

Some pull requests get no CodeRabbit review at all, and therefore cannot reach an
approval on their own:

- **Draft pull requests.** Mark the pull request ready for review.
- **Pull requests based on another feature branch.** CodeRabbit reviews only pull
  requests targeting the default branch. Retarget, or ask a human to review.

A code owner requirement is also outside CodeRabbit's reach. A GitHub App cannot
be listed in `CODEOWNERS`, so a pull request touching an owned path always needs
an approval from the owning team.

## Related pages

- [Agent policy](agent-policy.md)
- [Community health](community-health.md)
