# Routing Rules

## Label-Based Routing

| Pattern | Route To | Reason |
|---------|----------|--------|
| bug, fix, error | Tester | Quality/regression work |
| feature, implement, add | Developer | New functionality |
| security, auth, vulnerability | Reviewer | Security-sensitive work |
| research, investigate, explore | Researcher | Deep analysis needed |
| architecture, design, refactor | Lead | Architectural decisions |

## Full-Team Triggers

Issues labeled `squad` (without a member suffix) trigger full-team dispatch where ALL members analyze the issue from their role's perspective.

## Escalation

If no routing match is found, the issue is assigned to Lead for further triage.
