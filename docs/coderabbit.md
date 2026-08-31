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

To keep the shared settings and add your own on top, set `inheritance: true` and
list only what you want to change:

```yaml
inheritance: true
remote_config:
  repository: geolonia/.github
  ref: main
  path: .coderabbit.yaml
reviews:
  profile: chill
  path_filters:
    - "!vendor/**"
```

Without `inheritance: true`, any keys you add next to `remote_config` are ignored.

## Replace it

To opt out entirely, delete the `remote_config` block, delete `inheritance: true`
if you added it, and write a full configuration for your repository. Leaving
`inheritance` on would keep merging settings from the organization level. See the
[CodeRabbit configuration reference](https://docs.coderabbit.ai/reference/yaml-template).

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
