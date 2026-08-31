# Shared CodeRabbit Configuration

CodeRabbit reviews pull requests across Geolonia repositories. The shared review
settings live in one file in this repo:
[`.coderabbit.yaml`](https://github.com/geolonia/.github/blob/main/.coderabbit.yaml).

## Use the shared settings

Put this in `.coderabbit.yaml` at the root of your repository:

```yaml
remote_config:
  repository: geolonia/.github
  ref: main
  path: .coderabbit.yaml
```

That is the whole setup. Repositories created through the Backstage
`create-repository` template already have it.

## Extend it

You cannot, for now. A repository either uses the shared settings as they are, or
replaces them.

Two things that look like they should work, and do not:

- Adding keys next to `remote_config`. They are ignored.
- Adding `inheritance: true` as well. This is worse: the shared settings are
  dropped entirely and only your own keys survive, merged with the organization
  defaults. In particular `request_changes_workflow` reverts to `false`, so
  CodeRabbit stops approving pull requests in that repository and every pull
  request needs a human approver again. It fails silently.

If you need a setting changed, open a pull request against the shared file so
everyone gets it, or replace the configuration outright as below.

## Replace it

To opt out entirely, delete the `remote_config` block and write a full
configuration for your repository. See the
[CodeRabbit configuration reference](https://docs.coderabbit.ai/reference/yaml-template).

If you do this, your repository no longer gets shared settings, including the one
that lets CodeRabbit approve pull requests. Set `reviews.request_changes_workflow:
true` yourself, or every pull request there will need a human approver.

## Good to know

- A configuration change does not affect the pull request that makes it. Reviews
  use the configuration already on the base branch, so changes apply from the next
  review after they merge.
- Comment `@coderabbitai configuration` on any pull request to see the settings it
  resolved and where each came from.
- The shared file is public. Keep secrets out of it.

## Related pages

- [Agent policy](agent-policy.md)
- [Community health](community-health.md)
