# Project structure

```
.
├── profile/
│   └── README.md                         Org profile shown on github.com/geolonia
├── docs/                                 TechDocs source (MkDocs)
│   ├── index.md
│   ├── agent-policy.md                   Org-wide AI agent policy
│   ├── community-health.md
│   ├── context-maintenance.md
│   ├── profile.md
│   ├── roadmap-workflow.md
│   ├── tooling.md
│   └── workflows.md                      How the reusable workflows pattern works
├── workflow-templates/                   Templates shown in GitHub's "New workflow" UI
│   ├── publish-techdocs.yml
│   ├── publish-techdocs.properties.json
│   ├── cdk-deploy-monitor.yml
│   ├── cdk-deploy-monitor.properties.json
│   ├── release-auto-on-tag.yml
│   ├── release-auto-on-tag.properties.json
│   ├── sync-team-access.yml
│   ├── sync-team-access.properties.json
│   └── CODEOWNERS
├── .github/
│   ├── workflows/
│   │   ├── publish-techdocs.yml          This repo's own TechDocs publish entry
│   │   ├── release-auto-on-tag.yml       This repo's own release entry
│   │   ├── reusable-backstage-techdocs.yml      Logic - called by other repos
│   │   ├── reusable-cdk-deploy-monitor.yml      Logic - called by other repos
│   │   ├── reusable-release-auto-on-tag.yml     Logic - called by other repos
│   │   └── reusable-sync-team-access.yml        Logic - called by other repos
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.yml
│   │   ├── feature_request.yml
│   │   └── config.yml
│   ├── pull_request_template.md
│   └── CODEOWNERS                        This repo's CODEOWNERS
├── CODE_OF_CONDUCT.md                    Org-wide community health files -
├── CONTRIBUTING.md                          inherited by every Geolonia repo that
├── SUPPORT.md                               does not override locally
├── SECURITY.md
├── LICENSE
├── README.md
├── AGENTS.md                             Repo-local agent guidance
├── CLAUDE.md                             Points at AGENTS.md
├── catalog-info.yaml                     Backstage entity
├── .coderabbit.yaml                      CANONICAL source for the org -
│                                            referenced via remote_config: by
│                                            every other Geolonia repo
├── .markdownlint.json
├── .editorconfig
└── mkdocs.yml                            TechDocs build config
```

## Two locations for workflows - the key idea

GitHub's "New workflow" picker reads `workflow-templates/` (and its
`.properties.json` siblings). The actual reusable logic lives in
`.github/workflows/reusable-*.yml`. Templates in `workflow-templates/`
point at tagged versions (e.g. `@v1`) of the reusable workflows so
consumers pin a stable version. See the `reusable_workflows` memory.

## Build artefacts and ignored paths

This is a documentation/templates repo - no build artefacts. `.serena/`
is gitignored; the `.gitignore` keeps the tree minimal.
