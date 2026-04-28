# Tooling

This list covers tools used across Geolonia repositories.

## Required

- GitHub CLI (`gh`)

## Recommended

- OpenCode
- Claude Code
- Codex review

## Org-wide MCP servers

Geolonia operates a single read-only Backstage MCP server that exposes the
software catalog, the Scaffolder (read-only), and notifications as MCP
tools. Agents can use it to answer cross-repo catalog questions, look up
templates, and read notifications without leaving the editor.

- URL: `https://backstage.hub.geolonia.com/api/mcp-actions/v1/geolonia-backstage`
- Auth: per-user OAuth via your existing Backstage GitHub sign-in. No shared
  token. The first MCP call opens a browser approval popup; once approved
  the token is cached locally by your client.
- Read-only by design. Mutating actions (registering or unregistering
  entities, executing Scaffolder templates) still go through the Backstage
  UI or `catalog-info.yaml` PRs. Future writable actions will be exposed
  under separate endpoints so they remain explicit opt-ins.

Claude Code setup:

```bash
claude mcp add --transport http geolonia-backstage \
  https://backstage.hub.geolonia.com/api/mcp-actions/v1/geolonia-backstage
```

Tools exposed:

- `catalog.get-catalog-model-description`: markdown overview of catalog
  kinds, annotations, relations.
- `catalog.get-catalog-entity`: fetch a single entity by kind, namespace,
  and name.
- `catalog.query-catalog-entities`: predicate queries with full text,
  sort, and pagination.
- `catalog.validate-entity`: validate `catalog-info.yaml` contents.
- `scaffolder.list-scaffolder-actions`: list installed Scaffolder actions.
- `scaffolder.list-scaffolder-tasks`: list template tasks.
- `scaffolder.get-scaffolder-task-logs`: fetch log events for a task.
- `scaffolder.dry-run-template`: validate a template without making
  changes.
- `notifications.get-notifications`: fetch the signed-in user's
  notifications.

The server itself is implemented in `geolonia/geolonia-backstage` via the
experimental `@backstage/plugin-mcp-actions-backend`. Filter syntax and
auth model may change with upstream releases.
