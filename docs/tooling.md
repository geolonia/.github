# Tooling

This list covers tools used across Geolonia repositories.

## Required

- GitHub CLI (`gh`)

## Recommended

- OpenCode
- Claude Code
- Codex review

## Org-wide MCP servers

Geolonia runs an internal Backstage MCP server that AI assistants
(Claude Code, Cursor, etc.) can use to answer questions about and act on
our software catalog.

Endpoint, auth flow, available tools, and setup instructions live in the
internal Backstage TechDocs site (under "Using Backstage > MCP Server")
and in repository-level `AGENTS.md` files. Access is gated by per-user
OAuth against the Geolonia GitHub organization.
