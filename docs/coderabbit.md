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

Not possible today. Take the shared settings as they are, or replace them.

Adding your own keys next to `remote_config` does nothing. Adding
`inheritance: true` is worse: it throws the shared settings away without telling
you, and CodeRabbit stops approving pull requests in that repository.

If you want a setting changed, change it in the shared file so everyone gets it.

## Replace it

Delete everything in `.coderabbit.yaml` and write your own, using the
[CodeRabbit reference](https://docs.coderabbit.ai/reference/yaml-template). Include
`reviews.request_changes_workflow: true`, or CodeRabbit will not approve pull
requests and every one will need a human approver.

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
