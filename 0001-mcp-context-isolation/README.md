# RFC-0001: MCP Context Isolation

- **Status**: Draft
- **Author**: Lucas Wang (Claude World)
- **Created**: 2026-01-12
- **Updated**: 2026-01-12
- **Related Issues**: [#7336](https://github.com/anthropics/claude-code/issues/7336), [#11364](https://github.com/anthropics/claude-code/issues/11364), [#7172](https://github.com/anthropics/claude-code/issues/7172)

## Summary

Assign MCP servers to specific agents and skills via frontmatter declaration, loading them into **isolated forked contexts** instead of the main session context. This keeps the main context permanently clean while enabling full MCP functionality when needed.

## Motivation

### The Problem: Context Budget Under Pressure

With multiple MCP servers enabled, tool definitions consume significant context before any work begins:

| MCP Server | Approximate Token Cost | Source |
|------------|----------------------|--------|
| GitHub (27 tools) | ~18,000 | [Issue #11364](https://github.com/anthropics/claude-code/issues/11364) |
| AWS MCP servers | ~18,300 | [Issue #7172](https://github.com/anthropics/claude-code/issues/7172) |
| Cloudflare | ~15,000+ | Community reports |
| Sentry | ~14,000 | Community reports |
| Playwright (21 tools) | ~13,647 | [Scott Spence](https://scottspence.com/posts/optimising-mcp-server-context-usage-in-claude-code) |
| Supabase | ~12,000+ | Community reports |
| Average per tool | ~550-850 | Issue #11364 |

**Real-world impact**: With 7 MCP servers active, tool definitions alone consume **67,300 tokens (33.7% of 200k context)**. Even a minimal 3-server setup consumes 42,600 tokens (21.3%).

### The Modern Work Hub Dilemma

Modern knowledge workers manage numerous platforms:

| Category | Platforms |
|----------|-----------|
| Code Hosting | GitHub, GitLab, Bitbucket |
| Project Management | Jira, Linear, Asana, Notion |
| Communication | Slack, Discord, Teams |
| CI/CD | Vercel, Netlify, Cloudflare |
| Monitoring | Sentry, Datadog |

**The Dilemma:**
- **Option A (Install All)**: 50,000+ tokens consumed at session start = 50% context gone
- **Option B (Separate by Project)**: Defeats Claude Code's value as a unified command center

Neither option is acceptable for Claude Code to fulfill its potential as a **universal work orchestrator**.

## Detailed Design

### Key Distinction: Context Isolation, Not Lazy Loading

> **This proposal is fundamentally different from traditional lazy loading approaches.**

| Approach | Main Context | Load Time | Complexity |
|----------|-------------|-----------|------------|
| **Traditional Lazy Loading** | Gets populated when MCP needed | Runtime dynamic | High (state management) |
| **Our Proposal: Context Isolation** | **Always stays clean** | At fork creation | Low (reuses `context: fork`) |

```
Traditional Lazy Loading:
Main Context ──[need MCP]──> Load MCP ──> Main Context (now occupied)

Our Proposal (Context Isolation):
Main Context (stays clean)
    └── Fork Agent Context ──> Load MCPs ──> Isolated Context
                                              └── Released when done
```

### Proposed Architecture

#### Two-Sided Configuration

**1. MCP-Side: Lazy Loading Flag (settings.json)**

```json
{
  "mcpServers": {
    "memory": {
      "command": "...",
      "lazy": false    // Always load (default, backward compatible)
    },
    "github": {
      "command": "...",
      "lazy": true     // Don't load until requested
    },
    "postgres": {
      "command": "...",
      "lazy": true     // Don't load until requested
    }
  }
}
```

**Backward compatibility**: Omitting `lazy` or setting `lazy: false` maintains current behavior.

**2. Agent/Skill-Side: Frontmatter Declaration**

```yaml
# agents/database-specialist.md
---
name: database-specialist
description: Database operations expert
tools: [Read, Bash, Grep]
mcp:
  required: [postgres]    # Must have
  optional: [redis]       # Nice to have
context: fork
---
```

```yaml
# skills/deploy/SKILL.md
---
description: Deploy to production
mcp: [vercel, github]
context: fork
---
```

#### Loading Logic

| MCP `lazy` Setting | Agent/Skill Declaration | Result |
|-------------------|------------------------|--------|
| `false` (or omitted) | - | ✅ Load at session start (current behavior) |
| `true` | Not declared | ❌ Don't load |
| `true` | `mcp: [xxx]` | ✅ Load when agent/skill runs |

#### Result: Clean Main Context

```
Main Session (Lean)
    │
    ├── Base MCPs only: filesystem, memory
    │   (~1,300 tokens instead of 67,000+)
    │
    ├── Task: database-specialist (forked)
    │   └── Loads: postgres, redis (isolated)
    │
    └── Skill: /deploy (forked)
        └── Loads: vercel, github (isolated)
```

### Progressive Permission Layers

```
Layer 0: Main Context (minimal)
   └── filesystem (read-only), memory

Layer 1: Development Agents
   └── code-reviewer: + git (read)
   └── debugger: + bash (sandboxed)

Layer 2: Specialized Skills
   └── /deploy: + vercel, github (push)
   └── /db-migrate: + postgres (write)

Layer 3: Admin Operations
   └── /production-access: all (with confirmation)
```

## Benefits

1. **Context Efficiency**: Main context stays permanently clean (not temporarily)
2. **Granular Permissions**: Each agent/skill has its own MCP scope
3. **Progressive Security**: Layered access control instead of all-or-nothing
4. **Scalability**: Enable 20+ MCP servers without context pressure
5. **Low Implementation Cost**: Reuses existing `context: fork` infrastructure

## Alternatives Considered

### 1. Traditional Lazy Loading (Runtime Dynamic)

Load MCPs into main context when first used.

**Rejected because:**
- Main context still gets polluted
- Complex state management for unloading
- Doesn't solve the fundamental problem

### 2. Manual Server Management

Users manually enable/disable MCPs per session.

**Rejected because:**
- Disruptive to workflow
- Requires predicting tool needs in advance
- Requires session restart

### 3. Keyword-Based Triggers

Auto-load MCPs when certain keywords are mentioned.

**Rejected because:**
- Unreliable triggering
- Still loads into main context
- Complex to configure

## Challenges

| Challenge | Possible Solution |
|-----------|-------------------|
| MCP startup latency | Warm pool, pre-connect on first mention |
| State after fork ends | Stateless design, session-level cache |
| Tool discovery | Lazy manifest (tools declared but not loaded) |
| Credential scoping | Environment inheritance with limits |
| MCP returning huge data | Streaming, chunking, or size limits per fork |

## Prior Art

- **Claude Code `context: fork`** (2.1.0): Already provides context isolation for skills
- **Issue #7336**: Lazy loading feature request (43 upvotes)
- **[machjesusmoto/claude-lazy-loading](https://github.com/machjesusmoto/claude-lazy-loading)**: Proof-of-concept implementation
- **React Suspense**: Similar pattern of declaring dependencies and loading on demand

## Implementation Notes

This proposal leverages existing Claude Code infrastructure:

1. `context: fork` already creates isolated contexts
2. Agent/Skill frontmatter already supports `tools:` declarations
3. MCP configuration already exists in settings.json

The main additions:
- `lazy: true` flag in MCP configuration
- `mcp:` field in agent/skill frontmatter
- Logic to inject MCPs into forked contexts

## References

- [Full article on claude-world.com](https://claude-world.com/articles/mcp-lazy-loading)
- [Issue #7336: Lazy Loading Feature Request](https://github.com/anthropics/claude-code/issues/7336)
- [Issue #11364: Lazy-load MCP Tool Definitions](https://github.com/anthropics/claude-code/issues/11364)
- [Scott Spence: Optimising MCP Server Context Usage](https://scottspence.com/posts/optimising-mcp-server-context-usage-in-claude-code)

---

*This RFC is published by [Claude World](https://claude-world.com), a developer community focused on Claude Code best practices.*
