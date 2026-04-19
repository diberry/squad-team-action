# Squad Team Action

Run your full AI agent team on GitHub Actions. Label an issue with `squad` and watch your team analyze, discuss, and propose solutions—all in under 5 minutes.

## What This Does

Add the `squad` label to any GitHub issue. The action automatically:
- Clones your repo
- Runs your full Squad team on the issue  
- Posts results back as issue comments

**Works immediately on GitHub-hosted runners—no infrastructure setup required.**

```
Issue created → Add "squad" label → Team analyzes → Results posted
   (2–5 sec)        (instant)        (2–5 min)      (as comment)
```

Need it faster or cheaper at scale? We support **self-hosted runners on Azure Container Apps** (see below).

---

## Quick Start (GitHub-Hosted) ⚡

Get your team discussing issues in 5 minutes. No prerequisites—just a GitHub repo and a Copilot token.

### 1. Set up secrets

Go to **Settings → Secrets and variables → Actions** and add:

- **`COPILOT_TOKEN`** (required): Your Copilot personal access token
- **`COPILOT_ASSIGN_TOKEN`** (optional): For auto-assignment to @copilot

### 2. Define your team

Edit `.squad/team.md`:

```markdown
| Name | Role | Specialty |
|------|------|-----------|
| Alex | Architect | System design |
| Jordan | Developer | Implementation |
| Riley | DevRel | Documentation |
```

### 3. (Optional) Add routing rules

Edit `.squad/routing.md` to auto-assign members by keyword:

```markdown
| Keyword | Squad Member |
|---------|--------------|
| api | Jordan |
| docs | Riley |
| deploy | Alex |
```

### 4. Test it

1. Create an issue: "Add authentication to the API"
2. Add the `squad` label
3. Watch the workflows run (check **Actions** tab)
4. See results posted as a comment

**That's it.** Your team is now analyzing issues on every label.

---

## Customize Your Team

### `.squad/team.md` reference

This file defines who your team members are. Each row is one person:

```markdown
| Name | Role | Specialty |
|------|------|-----------|
| Your Name | Your Role | What you focus on |
```

Squad will use these when dispatching work. Edit it anytime—no redeploy needed.

### `.squad/routing.md` reference

(Optional) Route issues to specific team members based on keywords:

```markdown
| Keyword | Squad Member |
|---------|--------------|
| database | Alex |
| frontend | Jordan |
```

---

## Advanced: Self-Hosted Runners

### When to use self-hosted

- Running >100 squad analyses per month
- Need lower latency (10–15 sec vs 2–5 sec on GitHub-hosted)
- Cost-sensitive (free tier supports ~200 jobs/month)

### How it works

Self-hosted runners use **Azure Container Apps + KEDA**:

```
Issue labeled "squad"
    ↓
GitHub Actions detects [self-hosted, linux, squad] runner needed
    ↓
KEDA scales up ACA Job container
    ↓
Runner starts, picks up workflow, runs Squad
    ↓
Results posted, container exits
    ↓
KEDA scales down (no idle cost)
```

### Setting up self-hosted runners

**Prerequisites:**
- Azure subscription with permissions
- `az` CLI and Terraform 1.5+ (or use `azd`)

**Steps:**

1. **Deploy infrastructure** (from the [squad-on-aca](https://github.com/haflidif/squad-on-aca) repo):
   ```bash
   cd squad-on-aca
   azd up
   ```

2. **Add the same secrets** to this repo:
   - `COPILOT_TOKEN`
   - `COPILOT_ASSIGN_TOKEN` (optional)

3. **Update your workflow** to use self-hosted runners:
   ```yaml
   # Change from:
   runs-on: ubuntu-latest
   
   # To:
   runs-on: [self-hosted, linux, squad]
   ```

4. **Label issues as normal**—workflows now run on your ACA infrastructure.

### Cost comparison

| Runner | Cost/month (100 jobs × 2 min) | Setup | Cold start |
|--------|-------------------------------|-------|-----------|
| **GitHub-hosted** | ~$1.60 ($0.008/min) | None | 2–5 sec |
| **Self-hosted (ACA)** | ~$0–3 (free tier) | Terraform | 10–15 sec |

At 1,000 jobs/month (2 min each): GitHub-hosted ~$16 vs self-hosted ~$10–15. Check [current GitHub Actions pricing](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions) for your plan.

---

## Workflows Reference

| Workflow | File | Trigger | Purpose |
|----------|------|---------|---------|
| **Squad Full-Team** | `squad-full-team-dispatch.yml` | `squad` label | Runs full team dispatch; posts results |
| **Squad Triage** | `squad-triage.yml` | `squad` label | Routes by keyword; adds `squad:{member}` labels |
| **Squad Issue Assign** | `squad-issue-assign.yml` | `squad:{member}` label | Posts assignment; triggers @copilot if enabled |

The main workflow is **Squad Full-Team Dispatch**. Others support routing and fallback.

---

## Secrets Reference

Store these in **Settings → Secrets and variables → Actions**:

- **`COPILOT_TOKEN`** (required): Copilot personal access token (used by Squad CLI)
- **`COPILOT_ASSIGN_TOKEN`** (optional): PAT for @copilot agent (for auto-assignment)
- **`GITHUB_TOKEN`**: Provided automatically by GitHub Actions

---

<details>
<summary><b>Architecture Deep Dive</b> — How the two approaches compare</summary>

We built Squad Team Action on a **self-hosted GitHub Actions runner** model. For context, here's how it compares to the **queue-driven approach** from [haflidif/squad-on-aca](https://github.com/haflidif/squad-on-aca).

### Approach A: Queue-Driven (Hafliði's Squad-on-ACA)

**How it works:**
1. GitHub label → enqueues JSON message to Azure Storage Queue
2. KEDA `azure-queue` scaler polls the queue
3. ACA Job container runs custom entrypoint:
   - Clone repo, run Squad, create PR, delete queue message

**Strengths:**
- Simple: One container = one complete action
- PR creation built in (not orchestrated externally)
- Excellent cost at scale: ~$6/month for 100 issues
- Lightweight revision loop via `/squad revise` comments

**Trade-offs:**
- Custom container logic (not YAML)
- Single agent per container (harder for full-team dispatch)
- Queue schema couples to entrypoint (container rebuild on change)

### Approach B: Self-Hosted Runner (Squad Team Action)

**How it works:**
1. GitHub label → detects self-hosted runner needed
2. KEDA `github-runner` scaler detects pending GitHub Actions jobs
3. Spins up ACA Job container with GitHub Actions runner binary
4. Workflow YAML orchestrates dispatch → Squad → results
5. Container exits after workflow completes

**Strengths:**
- Workflow logic in YAML (easy iteration, no rebuild)
- Full-team dispatch via Squad CLI
- GitHub Actions ecosystem (caching, artifacts, matrix)
- Simpler mental model: "GitHub Actions on your own compute"
- Pre-baked tools: Node.js 20, Squad CLI, gh CLI, Copilot extension

**Trade-offs:**
- More infrastructure: KEDA, ACA, runner registration, OIDC
- 10–15 sec cold start (ACA Job spin-up)
- More Azure knowledge required

### Side-by-Side Comparison

| Aspect | Queue-Driven | Self-Hosted Runner |
|--------|--------------|-------------------|
| **What triggers ACA** | Storage Queue message | GitHub Actions job queue |
| **Container does** | Clone → Squad → PR (complete) | Runs GitHub Actions runner |
| **Orchestration** | Container entrypoint.sh | Workflow YAML |
| **KEDA scaler** | `azure-queue` | `github-runner` |
| **Team model** | Single agent per container | Full team dispatch |
| **Iteration** | Rebuild container | Edit YAML |
| **Cost (100 jobs/mo)** | ~$6 | ~$0 (free tier) |

### Infrastructure (Self-Hosted Only)

Shared Azure components:
- **Azure Container Apps Managed Environment** — hosts containers
- **Azure Container Registry (ACR)** — stores images
- **User-Assigned Managed Identity** — grants permissions (AcrPull, Key Vault)
- **Azure Key Vault** — stores secrets (Copilot PAT, GitHub App credentials)
- **Log Analytics Workspace** — logs and diagnostics
- **OIDC Federated Identity** — GitHub Actions auth without secrets

Self-hosted runner adds:
- **KEDA Scaler (github-runner)** — polls GitHub job queue, scales ACA Jobs
- **Runner container** — GitHub Actions runner binary + entrypoint
- **Terraform modules** — `infra/runner.tf` in [squad-on-aca](https://github.com/haflidif/squad-on-aca)

All infrastructure code lives in the `squad-on-aca` repo. Squad Team Action just defines the workflows.

</details>

---

## Limitations

- **Copilot CLI auth**: Requires valid `COPILOT_TOKEN`; falls back to label-based dispatch without it
- **Execution time**: Full-team dispatch takes 2–5 minutes depending on complexity
- **Rate limits**: Copilot CLI and GitHub API have limits; high-volume users may hit them
- **Cold start (self-hosted)**: ACA Jobs take 10–15 seconds vs 2–5 sec for GitHub-hosted

---

## Credits

- **[Hafliði Fridthjofsson](https://github.com/haflidif)** — [squad-on-aca](https://github.com/haflidif/squad-on-aca) and [blog post](https://azureviking.com/post/squad-on-aca-serverless-ai-agents/)
- **[Brady Gaster](https://github.com/bradygaster)** — Squad framework and Copilot CLI
- GitHub Actions and Azure Container Apps teams

---

## License

MIT
