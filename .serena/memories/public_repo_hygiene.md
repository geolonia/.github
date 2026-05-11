# Public-repo hygiene

**This repo (`geolonia/.github`) is public, and must remain public** for
GitHub's `.github`-repo special handling to work. Anything committed
here (code, docs, memories, workflows, templates) is world-readable
forever.

This memory codifies what must **not** appear in any change to this
repo, including in `.serena/memories/` files now that they are tracked.

## Do not include

- **Secrets** of any kind: API tokens, deploy keys, passwords, signed
  URLs, `op://` references that resolve to private values, AWS access
  keys, GitHub PATs.
- **AWS account IDs** (the 12-digit numbers), **ARNs that include
  account IDs**, **IAM role names** scoped to specific accounts,
  internal account aliases.
- **Internal hostnames or URLs**: any private subdomain, internal
  Backstage instance URLs, bastion / VPN hostnames, private S3
  bucket names.
- **References to private repos**:
  - Names of any sibling repo that is not itself public. Even an
    incidental mention confirms the repo exists.
  - Issue / PR numbers in private repos (`<org>/<private-repo>#NNN`)
    - these confirm the repo exists and reveal activity-level
    metadata.
  - File paths that imply internal workspace structure (parent
    submodule names, monorepo layout, etc.).
- **Customer data**: names, email addresses, support-ticket numbers,
  domains belonging to customers.
- **Staff personal data**: individual email addresses, phone numbers,
  Slack handles (group aliases like `group:geolonia/operations` are
  fine - they are already published in the org policy).
- **Internal-only opinions or critiques** of people, vendors, or
  partners.

## Allowed and useful

- Public GitHub URLs (`https://github.com/geolonia/...`) - but only
  for repos that are themselves public.
- Public org policy documents and their links
  (`docs/agent-policy.md`, etc.).
- Workflow and action names (the names themselves, not their secret
  values).
- Names of org secrets / variables (e.g. `TECHDOCS_AWS_ACCOUNT_ID` is
  fine as a label; its value is not).
- Patterns and conventions: PR / commit naming, branch strategies,
  code-style rules, markdownlint settings.
- Generic "how this org works" documentation - that is the whole point
  of this repo.

## When in doubt

- Rephrase to describe the **pattern** instead of the specific internal
  artifact. E.g. "tracked via an issue in our operations repo" rather
  than naming the repo and issue number.
- Strip cross-repo issue / PR links that point at private repos.
- Keep self-references to this very repo (`geolonia/.github#NN`) -
  those are public.
- If a sentence requires private context to make sense, the right move
  is usually to delete the sentence, not paraphrase it.
- Avoid naming private artifacts even in examples or grep patterns. A
  literal example string is a published string, regardless of the
  surrounding sentence saying "do not include this."

## Verification before opening a PR

Build a local grep against the names of repos, domains, and workspace
paths your org keeps private. Keep the pattern in a personal note or
shell function - **do not commit the pattern itself to this repo**, or
the published file becomes a directory of the very things it bans.

Sketch (replace each bracketed placeholder with the literal value you
want to catch):

```sh
# Private-repo / internal-path references in the staged diff:
git diff --cached | grep -E '<private-repo-name>|<workspace-path>|@<staff-domain>'

# Likely secrets:
git diff --cached | grep -iE 'secret|password|token|api[-_]?key|AKIA[0-9A-Z]{16}'

# AWS account IDs (12-digit numbers - inspect each hit in context):
git diff --cached | grep -E '\b[0-9]{12}\b'
```

A hit is not always a leak (the word "secret" appears in this very
memory), but every match deserves a deliberate look. See
`task_completion_checklist` step 4 for the same check in the standard
PR ritual.

## If a leak has already shipped

Treat anything previously committed to a public repo as compromised,
even if you delete it later. Git history preserves the value, and
crawlers / forks may have already cached it. Steps:

1. **Rotate** the secret / credential immediately.
2. Remove the value from the current tree and commit.
3. Decide whether `git filter-repo` history rewrite is worth the
   downstream pain (forks, clones, open PRs). For most non-credential
   leaks - e.g. a private repo name mentioned in a memory - rotation
   is not relevant and a forward fix is enough.
