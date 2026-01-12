---
description: Monitor application health and investigate issues
allowed-tools: [Read, Bash, Grep]
mcp:
  required: [sentry]
  optional: [datadog]
context: fork
---

# /monitor Skill

Monitor application health, investigate errors, and analyze performance.

## Capabilities

### Error Tracking (Sentry)
- List recent errors and exceptions
- Analyze error frequency and patterns
- Identify affected users
- Track error resolution status

### Performance Monitoring (Datadog)
- Check application metrics
- Analyze response times
- Monitor resource usage
- Review alerting status

## Common Tasks

### Investigate Recent Errors
```
/monitor errors --last 24h
```

### Check Error Trends
```
/monitor trends --period week
```

### Performance Analysis
```
/monitor performance --service api
```

### Create Issue from Error
```
/monitor create-issue --error-id xxx
```

## Workflow Example

1. Check Sentry for new errors
2. Analyze error stack traces
3. Correlate with Datadog metrics
4. Identify root cause
5. Create GitHub issue with findings

## MCP Usage

- **Sentry** (~14,000 tokens): Required for error tracking
- **Datadog** (variable): Optional for deeper metrics

By loading these MCPs only when `/monitor` is invoked, we save ~14,000+ tokens in normal sessions where monitoring isn't needed.
