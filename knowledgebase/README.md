# Anthropic Engineering Knowledge Base

This folder contains curated articles from Anthropic's engineering blog focused on building effective AI agents with Claude.

## Articles

| # | Article | Key Topics |
|---|---------|------------|
| 01 | [Effective Harnesses for Long-Running Agents](01-effective-harnesses-for-long-running-agents.md) | Session continuity, progress files, incremental development |
| 02 | [Advanced Tool Use](02-advanced-tool-use.md) | Tool search, programmatic calling, examples |
| 03 | [Code Execution with MCP](03-code-execution-with-mcp.md) | Code APIs, token efficiency, privacy |
| 04 | [Agent Skills](04-agent-skills.md) | Skill architecture, progressive disclosure, SKILL.md |
| 05 | [Context Engineering](05-context-engineering.md) | Attention budget, just-in-time loading, sub-agents |
| 06 | [Multi-Agent Research System](06-multi-agent-research-system.md) | Orchestrator-worker pattern, parallelization |
| 07 | [Claude Code Best Practices](07-claude-code-best-practices.md) | CLAUDE.md, workflows, headless mode |

## Quick Reference: Core Concepts

### Token Efficiency
- **Tool Search**: 85% reduction in token usage
- **Programmatic Tool Calling**: 37% reduction
- **Code APIs vs Direct Calls**: 98.7% reduction
- Token usage accounts for 80% of performance variance

### Architecture Patterns
- **Progressive Disclosure**: Load information in layers as needed
- **Orchestrator-Worker**: Lead agent delegates to specialized subagents
- **Just-in-Time Loading**: Maintain identifiers, load content dynamically

### Performance Gains
- Multi-agent systems outperform single agents by 90%+ on research tasks
- Parallel execution cuts research time by up to 90%
- Tool use examples improve accuracy from 72% to 90%

### Key Files
- **CLAUDE.md**: Configuration and context for Claude Code
- **SKILL.md**: Skill definition with name/description frontmatter
- **init.sh**: Session startup script for long-running agents
- **claude-progress.txt**: Progress tracking across sessions

### Thinking Modes
Use for complex problems requiring more computation:
- "think"
- "think hard"
- "think harder"
- "ultrathink"

## Sources

All articles from: https://www.anthropic.com/engineering
