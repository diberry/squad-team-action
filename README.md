# Squad Autonomous GitHub Action

**Architecture #2** — GitHub Actions native Squad dispatch. When the `squad` label is added to an issue, the full AI agent team is dispatched on the GitHub Actions runner to analyze and respond.

## What This Does

This repo provides a GitHub Actions workflow that bridges the gap between Squad's label-based routing and actual AI agent execution. Existing Squad workflows manage labels and comments — this one **runs the full team**.

```
┌─────────────────────────────────────────────────────────────────┐
│                     GitHub Issue                                │
│                                                                 │
│  User adds "squad" label                                        │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────────────────┐                               │
│  │  squad-full-team-dispatch.yml │ ◄── The key innovation       │
│  │  (GitHub Actions runner)      │                               │
│  └──────────┬───────────────────┘                               │
│             │                                                    │
│             ├── Install Squad CLI + Copilot CLI                  │
│             ├── Fetch issue details                              │
│             ├── Post "processing" comment                        │
│             ├── Run full-team dispatch via Copilot CLI           │
│             │   (or fallback: add squad:{member} labels)         │
│             └── Post results back to issue                       │
│                                                                 │
│  ┌──────────────────┐    ┌────────────────────────┐             │
│  │ squad-triage.yml  │    │ squad-issue-assign.yml  │            │
│  │ (keyword routing) │    │ (@copilot assignment)   │            │
│  └──────────────────┘    └────────────────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works

1. **Trigger**: A `squad` label is added to a GitHub issue (or manual `workflow_dispatch`)
2. **Setup**: The runner installs Squad CLI and GitHub Copilot CLI
3. **Dispatch**: The full team analyzes the issue from each role's perspective
4. **Results**: Analysis is posted back as issue comments
5. **Fallback**: If Copilot CLI auth isn't available, adds `squad:{member}` labels for each team member, triggering individual assignment workflows

## Setup

### 1. Fork or clone this repo

```bash
git clone <this-repo-url>
cd squad-autonomous-github-action-dfb
```

### 2. Configure secrets

In your repo's **Settings → Secrets and variables → Actions**, add:

| Secret | Required | Description |
|--------|----------|-------------|
| `COPILOT_TOKEN` | Yes | Personal access token with Copilot access for Squad CLI auth |
| `COPILOT_ASSIGN_TOKEN` | Optional | PAT for @copilot coding agent auto-assignment |

The built-in `GITHUB_TOKEN` is used automatically for issue comments, labels, and basic API calls.

### 3. Customize your team

Edit `.squad/team.md` with your actual team members and roles:

```markdown
| Name | Role | Specialty |
|------|------|-----------|
| Alice | Team Lead | Architecture, coordination |
| Bob | Developer | Feature implementation |
| ...  | ...       | ...                       |
```

Edit `.squad/routing.md` to customize how issues are routed to specific members based on keywords.

### 4. Use it

1. Create an issue in this repo
2. Add the `squad` label
3. Watch the full team analyze the issue and post results

You can also trigger manually via **Actions → Squad Full-Team Dispatch → Run workflow**.

## Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `squad-full-team-dispatch.yml` | `squad` label or manual | **Runs full AI team on the runner** |
| `squad-triage.yml` | `squad` label | Keyword-based routing to individual agents |
| `squad-issue-assign.yml` | `squad:{member}` label | Posts assignment comment, optionally assigns @copilot |

## Limitations & Future Work

- **Copilot CLI auth**: The runner needs a `COPILOT_TOKEN` with Copilot access. Without it, the workflow falls back to label-dispatch mode.
- **Execution time**: Full-team dispatch may take several minutes depending on issue complexity.
- **Rate limits**: Copilot CLI and GitHub API have rate limits that may affect large-scale usage.
- **Future**: Direct Squad SDK integration (bypass CLI), streaming results, per-agent comments, webhook-based event bus.

## License

MIT
