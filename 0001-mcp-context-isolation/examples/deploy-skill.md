---
description: Deploy to production with safety checks
allowed-tools: [Read, Bash, Grep]
mcp: [vercel, github, slack]
context: fork
---

# /deploy Skill

Deploy the current branch to production with comprehensive safety checks.

## Pre-deployment Checks

1. **Build Verification**
   - Run `pnpm build` and ensure no errors
   - Check for TypeScript errors
   - Verify all tests pass

2. **Branch Status**
   - Confirm on correct branch (main/master)
   - Check for uncommitted changes
   - Verify branch is up to date with remote

3. **GitHub Status**
   - All CI checks passing
   - No blocking reviews
   - No merge conflicts

## Deployment Steps

1. Create deployment on Vercel
2. Wait for build completion
3. Run smoke tests on preview URL
4. Promote to production
5. Notify team on Slack

## Rollback Plan

If deployment fails:
1. Identify the issue from Vercel logs
2. Revert to previous deployment
3. Notify team with error details

## Usage

```
/deploy                    # Deploy current branch
/deploy --preview          # Deploy to preview only
/deploy --skip-tests       # Skip test verification (not recommended)
```

## MCP Usage

This skill loads `vercel`, `github`, and `slack` MCPs in an isolated context. These heavy MCPs (~35,000+ tokens combined) don't occupy the main context.
