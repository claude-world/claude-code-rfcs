# Claude Code RFCs

Community-driven architectural proposals for Claude Code improvements.

## What is an RFC?

RFC (Request for Comments) is a process for proposing and discussing significant changes to Claude Code. This repository hosts proposals from the [Claude World](https://claude-world.com) community.

## Active RFCs

| RFC | Title | Status | Languages | Discussion |
|-----|-------|--------|-----------|------------|
| [0001](./0001-mcp-context-isolation/) | MCP Context Isolation | Draft | [EN](./0001-mcp-context-isolation/README.md) / [中文](./0001-mcp-context-isolation/README.zh-tw.md) / [日本語](./0001-mcp-context-isolation/README.ja.md) | [GitHub Issue #17668](https://github.com/anthropics/claude-code/issues/17668) |

## RFC Resources

Each RFC may include:

| Resource | Description |
|----------|-------------|
| `README.md` | Main proposal (English) |
| `README.<lang>.md` | Translations |
| `diagrams/` | Architecture diagrams (Mermaid) |
| `examples/` | Implementation examples |

### RFC-0001 Resources

- **[Diagrams](./0001-mcp-context-isolation/diagrams/)**: Visual architecture explanations
  - Context Isolation vs Traditional Lazy Loading
  - Loading Logic Flowchart
  - Permission Layers
  - Architecture Overview
- **[Examples](./0001-mcp-context-isolation/examples/)**: Sample configurations
  - settings.json with lazy flags
  - Agent frontmatter examples
  - Skill frontmatter examples

## RFC Process

```
Draft → Discussion → Submitted → Accepted/Rejected → Implemented
```

1. **Draft**: Initial proposal with problem statement, proposed solution, and implementation details
2. **Discussion**: Community feedback via GitHub Issues and Discord
3. **Submitted**: Formally submitted to Anthropic via GitHub Issue
4. **Accepted/Rejected**: Official response from Anthropic team
5. **Implemented**: Merged into Claude Code

## How to Contribute

See [CONTRIBUTING.md](./CONTRIBUTING.md) for detailed guidelines.

Quick start:
1. **Discuss first**: Join our [Discord](https://discord.gg/claude-world) to discuss ideas
2. **Write RFC**: Fork this repo, use [template.md](./template.md)
3. **Submit PR**: Open a pull request for community review
4. **Iterate**: Refine based on feedback

## RFC Template

See [template.md](./template.md) for the standard RFC format.

## About Claude World

[Claude World](https://claude-world.com) is a developer community focused on Claude Code best practices and advanced usage patterns. We build tools, share knowledge, and collaborate with the global Claude ecosystem.

- 🌐 Website: [claude-world.com](https://claude-world.com)
- 💬 Discord: [Claude World Taiwan](https://discord.gg/claude-world)
- 🐙 GitHub: [claude-world](https://github.com/claude-world)

## Related Links

- [Claude Code Official Repo](https://github.com/anthropics/claude-code)
- [MCP (Model Context Protocol)](https://modelcontextprotocol.io/)
- [Claude Code Documentation](https://docs.anthropic.com/claude-code)

## License

This repository is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). You are free to share and adapt the content with attribution.
