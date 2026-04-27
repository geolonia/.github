# Tooling

This list covers tools used across Geolonia repositories.

## Required

- GitHub CLI (`gh`)

## Recommended

- OpenCode
- Claude Code
- Codex review

## Org-wide MCP servers

Geolonia operates a Backstage MCP server that exposes the software catalog
(services, APIs, ownership, dependencies, lifecycle) as MCP tools. Agents
can use it to answer cross-repo catalog questions without leaving the
editor.

- URL: `https://backstage.hub.geolonia.com/api/mcp-actions/v1/catalog-readonly`
- Auth: per-user OAuth via your existing Backstage GitHub sign-in. No shared
  token. The first MCP call opens a browser approval popup; once approved
  the token is cached locally by your client.
- Read-only by design. Mutating actions (registering or unregistering
  entities) still go through the Backstage UI or `catalog-info.yaml` PRs.

Claude Code setup:

```bash
claude mcp add --transport http geolonia-backstage \
  https://backstage.hub.geolonia.com/api/mcp-actions/v1/catalog-readonly
```

Tools exposed:

- `catalog.get-catalog-model-description`: markdown overview of catalog
  kinds, annotations, relations.
- `catalog.get-catalog-entity`: fetch a single entity by kind, namespace,
  and name.
- `catalog.query-catalog-entities`: predicate queries with full text,
  sort, and pagination.
- `catalog.validate-entity`: validate `catalog-info.yaml` contents.

The server itself is implemented in `geolonia/geolonia-backstage` via the
experimental `@backstage/plugin-mcp-actions-backend`. Filter syntax and
auth model may change with upstream releases.
