# Copilot Instructions

This repository contains **Squad Team Action** — GitHub Actions workflows that orchestrate an AI agent team to autonomously triage and process GitHub issues.

## Repository structure

```
.github/workflows/
  squad-triage.yml         # Routes "squad"-labeled issues to a team member by keyword matching
  squad-full-team-dispatch.yml  # Dispatches all squad members to analyze an issue
  squad-issue-assign.yml   # Fires when a squad:{member} label is applied
.squad/
  team.md                  # Team roster (names, roles, slugs)
  routing.md               # Keyword → role routing rules
```

## How the workflows connect

1. A user labels an issue with `squad`.
2. **squad-triage** reads the issue title + body, matches keywords, and adds a `squad:{role}` label.
3. **squad-issue-assign** fires on the `squad:{role}` label, posts an assignment comment, and optionally assigns `@copilot` for developer roles.
4. **squad-full-team-dispatch** can also fire on the `squad` label (or via `workflow_dispatch`) to fan out labels to *all* team members at once.

## Secrets

| Secret | Required | Purpose |
|--------|----------|---------|
| `GITHUB_TOKEN` | Auto-provided | Issue/label operations |
| `COPILOT_TOKEN` | Optional | PAT with Copilot access for AI-powered dispatch |

## Conventions

- All `${{ }}` expressions that touch user-controlled data (issue titles, bodies, labels) are passed through `env:` blocks — never interpolated directly into `run:` scripts.
- Multiline values use `<<GHEOF` / `GHEOF` delimiters in `GITHUB_OUTPUT`.
- Labels are created on-demand with `gh label create ... 2>/dev/null || true` before applying.
- Workflows run on `ubuntu-latest`. No self-hosted runner dependency.

## When editing these workflows

- Keep shell injection prevention: pass GitHub context via `env:`, read from `$ENV_VAR` in shell.
- Test locally with `act` or push to a fork and trigger with a test issue.
- If adding new team members, update `.squad/team.md` — the dispatch workflow parses it automatically.
