---
description: Unified work management hub - orchestrate across all platforms
allowed-tools: [Read, Bash, Grep, Glob]
mcp:
  code:
    required: [github]
    optional: [gitlab]
  projects:
    required: [linear]
    optional: [jira, notion]
  communication:
    optional: [slack, discord]
  monitoring:
    optional: [sentry]
context: fork
---

# /work-hub Skill

A unified command center for orchestrating work across multiple platforms.

## The Vision

Instead of switching between 10+ browser tabs, manage everything from one place:

```
You: "Check my open PRs, update the Linear ticket, and notify the team"

/work-hub:
  1. Fetches open PRs from GitHub
  2. Updates Linear ticket status
  3. Posts summary to Slack
  4. All in one command
```

## Capabilities

### Cross-Platform Queries
- "What's blocking the release?"
- "Show all my open items across GitHub and Linear"
- "What did the team ship this week?"

### Automated Workflows
- PR merged → Update Linear → Notify Slack
- Error spike → Create issue → Assign on-call
- Deploy complete → Update docs → Announce

### Daily Standup Helper
- Summarize yesterday's merged PRs
- List today's priorities from Linear
- Flag blocked items

## Example Commands

```bash
# Morning briefing
/work-hub briefing

# Check release blockers
/work-hub blockers --release v2.0

# End-of-day summary
/work-hub summary --post-to slack

# Cross-platform search
/work-hub search "authentication bug"
```

## MCP Categories

| Category | MCPs | Purpose |
|----------|------|---------|
| Code | github, gitlab | PR reviews, commits, branches |
| Projects | linear, jira, notion | Tickets, docs, planning |
| Communication | slack, discord | Notifications, updates |
| Monitoring | sentry | Error correlation |

## Why Context Isolation Matters Here

Without context isolation, loading all these MCPs would consume:
- GitHub: ~18,000 tokens
- Linear: ~6,200 tokens
- Slack: ~3,000 tokens
- Sentry: ~14,000 tokens
- **Total: ~41,200 tokens (20%+ of context)**

With context isolation, these load only when `/work-hub` is invoked, and release when done.

## The Universal Orchestrator Dream

This skill demonstrates Claude Code's potential as a **universal work orchestrator**:
- One interface for everything
- Context-aware automation
- Cross-platform intelligence
- Zero context overhead when not in use
