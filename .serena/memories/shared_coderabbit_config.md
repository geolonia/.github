# Shared CodeRabbit config - this repo is the canonical source

Every Geolonia repo's CodeRabbit review settings come from
[`.coderabbit.yaml`](../../.coderabbit.yaml) in **this repo**. Other repos
contain only a 2-line pointer:

```yaml
remote_config:
  url: https://raw.githubusercontent.com/geolonia/.github/main/.coderabbit.yaml
```

## Implications when editing `.coderabbit.yaml` here

- **Effect is org-wide and immediate.** CodeRabbit fetches the URL at
  review time, so the next PR opened in any Geolonia repo uses the new
  config. There is no per-repo bump needed.
- **Don't add repo-specific settings here** (e.g. paths only meaningful
  to one repo). If a repo needs different rules, it drops the pointer
  and writes its own full `.coderabbit.yaml`.
- **No secrets.** The file is publicly fetchable by design.
- **Test impact carefully.** A bad change here breaks reviews everywhere
  on the next PR. Prefer small, well-justified changes; revert quickly
  if anything looks off.

## What it currently configures

- `language: en-US` - English reviews, no Japanese fallback.
- `reviews.profile: assertive`
- `reviews.auto_review.enabled: true` + `auto_incremental_review: true`
  (every push auto-retriggers).
- `reviews.path_filters` - skips `pnpm-lock.yaml`, `yarn.lock`,
  `package-lock.json`, `dist/`, `cdk.out/`, `coverage/`, `*.min.{js,css}`.
- `reviews.tools` - shellcheck, yamllint, actionlint, languagetool,
  markdownlint, hadolint.
- `chat.auto_reply: true`.

## Why not the dedicated `coderabbit` repo?

CodeRabbit's "Central Configuration" feature is hardcoded to a repo
named `coderabbit`. We chose `.github` over creating a new repo because
the file already lived here, no Backstage scaffolder run is needed, and
the trade-off (each new Geolonia repo has to add the 2-line pointer
explicitly, vs. auto-applied via a `coderabbit` repo) is acceptable.

The "shared configuration is not recommended" warning in the CodeRabbit
docs concerns secret exposure, which doesn't apply here - our shared
config is purely review-tool toggles.
