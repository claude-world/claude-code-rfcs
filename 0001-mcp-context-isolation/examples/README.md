# Implementation Examples

This directory contains example configurations demonstrating how MCP Context Isolation would work in practice.

## Contents

| File | Description |
|------|-------------|
| [settings.json](./settings.json) | MCP configuration with lazy loading flags |
| [database-specialist.md](./database-specialist.md) | Agent with database MCPs |
| [code-reviewer.md](./code-reviewer.md) | Agent with git/github MCPs |
| [deploy-skill.md](./deploy-skill.md) | Skill with deployment MCPs |
| [monitoring-skill.md](./monitoring-skill.md) | Skill with monitoring MCPs |
| [work-hub-skill.md](./work-hub-skill.md) | Complex skill with multiple MCP categories |

## Quick Start

1. **Configure MCPs as lazy** in `settings.json`:
   ```json
   {
     "mcpServers": {
       "github": { "command": "...", "lazy": true }
     }
   }
   ```

2. **Declare MCPs in agents/skills**:
   ```yaml
   ---
   name: my-agent
   mcp: [github]
   context: fork
   ---
   ```

3. **MCPs load only when the agent/skill runs**

## Usage Patterns

### Pattern 1: Simple MCP List

```yaml
mcp: [postgres, redis]
```

### Pattern 2: Required vs Optional

```yaml
mcp:
  required: [postgres]    # Must be configured
  optional: [redis]       # Use if available
```

### Pattern 3: Categorized (for complex skills)

```yaml
mcp:
  database: [postgres, redis]
  messaging: [slack, discord]
  monitoring: [sentry]
```
