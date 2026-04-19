# Squad Team Action

Run your full AI agent team directly on GitHub Actions. Add a `squad` label to any issue and the entire team analyzes, discusses, and proposes solutions — with support for both GitHub-hosted and self-hosted (Azure Container Apps) runners.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                   GITHUB-HOSTED (default)                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Issue labeled "squad"                                           │
│       │                                                          │
│       ▼                                                          │
│  GitHub Actions runner (ubuntu-latest)                           │
│       │                                                          │
│       ├── Clone repo                                             │
│       ├── Install Squad CLI + Copilot CLI extension              │
│       ├── Fetch issue details                                    │
│       ├── Run: copilot --yolo --agent squad                      │
│       │   (full-team dispatch via Squad framework)               │
│       └── Post results as issue comment                          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│            SELF-HOSTED (Azure Container Apps + KEDA)             │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Issue labeled "squad"                                           │
│       │                                                          │
│       ▼                                                          │
│  GitHub Actions detects self-hosted runner needed                │
│  [self-hosted, linux, squad]                                     │
│       │                                                          │
│       ▼                                                          │
│  KEDA GitHub Runner Scaler                                       │
│       │                                                          │
│       ├── Detect pending jobs                                    │
│       ├── Trigger Azure Container App Job                        │
│       │                                                          │
│       ▼                                                          │
│  ACA Job Container (ephemeral)                                   │
│       │                                                          │
│       ├── Start GitHub Actions runner binary                     │
│       ├── Runner registers with [self-hosted, linux, squad]      │
│       ├── Runner picks up workflow job                           │
│       ├── Clone repo                                             │
│       ├── Run Squad dispatch                                     │
│       └── Upload results                                         │
│       │                                                          │
│       ▼                                                          │
│  Container exits → Job completes                                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Why Self-Hosted Runners?

While **GitHub-hosted runners work out of the box**, self-hosted runners on Azure Container Apps offer:

| Factor | GitHub-hosted | Self-hosted (ACA) |
|--------|---------------|-------------------|
| **Cost** | $0.008/min (~$11.52/hour) | $0/mo (<200 jobs), ~$10–12/mo (1K jobs) |
| **Cold start** | ~2–5 sec | ~10–15 sec (ACA Jobs are ephemeral) |
| **Team size limit** | None | Limited by container quota |
| **Setup** | Works immediately | Requires Terraform + Azure |
| **Best for** | Small teams, prototyping | High-volume squads, cost-sensitive orgs |

**Pick GitHub-hosted by default.** Use self-hosted when you have >100 squad runs/month or need tighter cost control.

---

## Architecture Comparison: Two Approaches

We built Squad Team Action on two distinct architectural patterns. Understanding the trade-offs helps you choose the right tool.

### Approach A: Queue-Driven (Hafliði's Squad-on-ACA)

**How it works:**
1. GitHub label → enqueues JSON message to Azure Storage Queue
2. KEDA `azure-queue` scaler polls the queue
3. ACA Job container runs custom entrypoint:
   - Clone repo
   - Run `copilot --yolo --agent squad`
   - Create branch, push, open PR
   - Delete queue message

**Reference:** [Hafliði Fridthjofsson's blog post](https://azureviking.com/post/squad-on-aca-serverless-ai-agents/) • [GitHub: haflidif/squad-on-aca](https://github.com/haflidif/squad-on-aca)

**Strengths:**
- Simple: Container IS the entire workflow (one container = one agent action)
- Complete automation: PR creation baked in, not orchestrated externally
- Revision loop via `/squad revise` comments (lightweight, idempotent)
- Excellent cost at scale: ~$6/month for 100 issues

**Trade-offs:**
- Custom container entrypoint logic (not in YAML)
- Single agent per container (harder to do full-team dispatch)
- Queue message schema couples to entrypoint (changes require container rebuild)

### Approach B: Self-Hosted Runner (Squad Team Action)

**How it works:**
1. GitHub label → detects self-hosted runner needed
2. KEDA `github-runner` scaler detects pending GitHub Actions jobs
3. Spins up ACA Job container running GitHub Actions runner binary
4. Workflow YAML orchestrates the dispatch:
   - Squad CLI handles full-team dispatch internally
   - Runs: `copilot --yolo --agent squad`
   - Posts results as comments
5. Container exits after workflow completes

**Strengths:**
- Workflow logic lives in YAML (easy iteration, no container rebuild)
- Full-team dispatch: Squad CLI handles fan-out per issue
- GitHub Actions ecosystem: Reuse actions, caching, artifacts, matrix strategies
- Simpler mental model: "It's just GitHub Actions on your own compute"
- Pre-baked tools in runner image: Node.js 20, Squad CLI, gh CLI, Copilot extension
- Complementary to queue-driven approach (can run BOTH)

**Trade-offs:**
- More infrastructure: KEDA, ACA, runner registration, OIDC federation
- Workflow job waits for container to spin (10–15 sec cold start)
- Requires more Azure knowledge to troubleshoot

### Side-by-Side Comparison

| Aspect | Queue-Driven (Hafliði) | Self-Hosted Runner (Ours) |
|--------|----------------------|--------------------------|
| **What triggers ACA** | Storage Queue message | GitHub Actions job queue |
| **Container responsibility** | Clone → Squad → PR (complete) | Just run GitHub Actions runner |
| **Orchestration lives in** | Container entrypoint.sh | Workflow YAML |
| **KEDA scaler** | `azure-queue` | `github-runner` |
| **Agent model** | Single agent per container | Full team dispatch per workflow |
| **Workflow iteration** | Requires container rebuild | Just edit YAML |
| **Revision loop** | Built into entrypoint | Via workflow re-trigger |
| **Cost (100 jobs/mo)** | ~$6 | ~$0 (free tier) |
| **Container expertise** | More required | Less required |

**Recommendation:**  
→ **Queue-driven** if you want a fully self-contained container that handles clone-to-PR  
→ **Self-hosted runner** if you want workflow logic in YAML and easy iteration  
→ **Both** if your org does both single-agent and full-team work

---

## Infrastructure

Both approaches share common Azure infrastructure. Self-hosted runners add KEDA and runner configuration on top.

### Shared (both approaches)

- **Azure Container Apps Managed Environment** — hosts the containers
- **Azure Container Registry (ACR)** — stores container images
- **User-Assigned Managed Identity (UAMI)** — grants containers permissions (AcrPull, Key Vault reader)
- **Azure Key Vault** — stores secrets (Copilot PAT, GitHub App credentials)
- **Log Analytics Workspace** — container logs and diagnostics
- **OIDC Federated Identity** — GitHub Actions can authenticate to Azure without secrets

### Self-Hosted Runner Only

- **KEDA Scaler (github-runner)** — polls GitHub Actions job queue, scales ACA Jobs up/down
- **Runner container** — Dockerfile + entrypoint that runs GitHub Actions runner binary
- **Terraform modules** — `infra/runner.tf` in the main [squad-on-aca](https://github.com/haflidif/squad-on-aca) repo

All Terraform is in the main `squad-on-aca` repo. Squad Team Action just defines the workflows.

---

## Quick Start

### Option 1: GitHub-Hosted (Easiest)

No setup needed. The workflows work out of the box:

1. **Fork or clone this repo:**
   ```bash
   git clone https://github.com/your-org/squad-team-action
   cd squad-team-action
   ```

2. **Add secrets to your repo** (Settings → Secrets and variables → Actions):

   | Secret | Required | Description |
   |--------|----------|-------------|
   | `COPILOT_TOKEN` | Yes | Personal access token with Copilot access |
   | `COPILOT_ASSIGN_TOKEN` | Optional | PAT for @copilot auto-assignment |

3. **Customize your team:**
   - Edit `.squad/team.md` with your team members and roles
   - Edit `.squad/routing.md` with keyword-based routing rules

4. **Use it:**
   ```bash
   # Create an issue
   # Add the "squad" label
   # Watch workflows run on ubuntu-latest
   ```

**Cost:** ~$0.008/minute × number of workflow minutes/month  
**Performance:** Starts in 2–5 seconds

---

### Option 2: Self-Hosted (Azure Container Apps)

**Prerequisites:**
- Azure subscription with permissions to create resources
- `az` CLI installed and authenticated
- Terraform 1.5+ (or use `azd` if deployed via squad-on-aca repo)

**Steps:**

1. **Deploy infrastructure** (or reuse existing squad-on-aca deployment):
   ```bash
   # In the squad-on-aca repo, run:
   azd up
   # or use Terraform directly to apply infra/runner.tf
   ```

2. **In this repo, add the same secrets:**
   - `COPILOT_TOKEN`
   - `COPILOT_ASSIGN_TOKEN` (optional)

3. **Update one line in your workflow:**
   ```yaml
   # Change from:
   runs-on: ubuntu-latest
   
   # To:
   runs-on: [self-hosted, linux, squad]
   ```

4. **Label issues as before:**
   ```bash
   # Add "squad" label to any issue
   # Workflow will now run on self-hosted runner (ACA Job)
   ```

**Cost:** ~$0–12/month depending on job volume (free tier up to ~200 jobs)  
**Performance:** Starts in 10–15 seconds (ACA Job spin-up)

---

## Workflows

| Workflow | File | Trigger | Purpose |
|----------|------|---------|---------|
| **Squad Full-Team Dispatch** | `squad-full-team-dispatch.yml` | `squad` label or manual | Runs the full team dispatch; posts results as issue comments |
| **Squad Triage** | `squad-triage.yml` | `squad` label | Keyword-based routing; adds `squad:{member}` labels for each matching member |
| **Squad Issue Assign** | `squad-issue-assign.yml` | `squad:{member}` label | Posts assignment comment; optionally triggers @copilot auto-assignment |

The main workflow is **Squad Full-Team Dispatch**. The others support routing and assignment fallback.

---

## Configuration

### `.squad/team.md`

Define your team members and their roles:

```markdown
| Name | Role | Specialty |
|------|------|-----------|
| Alex | Architect | System design, tradeoffs |
| Jordan | Developer | Feature implementation, testing |
| Riley | DevRel | Docs, examples, DX |
```

### `.squad/routing.md`

Keyword-based routing for triage (optional):

```markdown
| Keyword | Squad Member |
|---------|--------------|
| api | Jordan |
| docs | Riley |
| deploy | Alex |
```

### Secrets

Store in **Settings → Secrets and variables → Actions**:

- **`COPILOT_TOKEN`** (required): Personal access token with Copilot access (used by Squad CLI)
- **`COPILOT_ASSIGN_TOKEN`** (optional): PAT for @copilot agent (if using auto-assignment)
- **`GITHUB_TOKEN`**: Provided automatically by GitHub Actions

---

## Cost Comparison

Real-world estimates for 100 squad runs/month (each run ~2 min):

| Runner Type | Monthly Cost | Notes |
|-------------|-------------|-------|
| GitHub-hosted | ~$23.04 | 100 runs × 2 min × $0.008/min × 144/month |
| Self-hosted (ACA) | ~$0–3 | Free tier up to ~200 runs; after that, compute costs apply (~$0.01–0.03/run) |

**At 1,000 runs/month:**
- GitHub-hosted: ~$230
- Self-hosted: ~$10–15

---

## Limitations & Future Work

### Current Limitations

- **Copilot CLI auth**: Workflows require a valid `COPILOT_TOKEN`. Without it, falls back to label-based dispatch.
- **Execution time**: Full-team dispatch may take 2–5 minutes depending on complexity.
- **Rate limits**: Copilot CLI and GitHub API have rate limits; large-scale users may hit them.
- **Self-hosted cold start**: ACA Jobs take 10–15 seconds to spin up (vs 2–5 sec for GitHub-hosted).

### Future Directions

- **Direct SDK integration**: Use Squad SDK directly instead of CLI (faster, richer telemetry)
- **Streaming results**: Post incremental results as each agent finishes
- **Per-agent comments**: Each agent's response in a separate comment thread
- **Webhook-based dispatch**: Alternate trigger model for non-GitHub sources
- **Cost optimization**: Spot instances for ACA Jobs (further reduce self-hosted cost)

---

## Credits

- **[Hafliði Fridthjofsson](https://github.com/haflidif)** — [squad-on-aca](https://github.com/haflidif/squad-on-aca) and [blog post](https://azureviking.com/post/squad-on-aca-serverless-ai-agents/) that inspired this approach
- **[Brady Gaster](https://github.com/bradygaster)** — Squad framework and Copilot CLI extension
- GitHub Actions and Azure Container Apps teams for excellent platform support

---

## License

MIT
