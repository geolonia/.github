# Context Maintenance Workflow

This workflow keeps org context accurate and lightweight across repositories.

## When to update context

- A user-facing behavior changes.
- A new domain term is introduced.
- A major decision or assumption is made.
- A workflow or process changes.

## What to update

- Repo-specific `AGENTS.md` for local instructions only.
- Org-level policy for rules that apply to all repos.
- Decision logs where they exist (repo-specific).

## Minimal checklist

- Confirm the change is reflected in docs or specs.
- Keep entries short and scannable.
- Link to the most relevant file or issue when possible.
