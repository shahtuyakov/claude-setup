# Agent Skills: Equipping AI Agents for Real-World Tasks

Source: https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills

## Overview

Anthropic has introduced **Agent Skills**, a new framework for building specialized AI agents. As stated in the article, "Claude is powerful, but real work requires procedural knowledge and organizational context."

Skills are organized directories containing instructions, scripts, and resources that agents can dynamically load to perform specialized tasks. Rather than building separate custom agents for each use case, developers can now equip general-purpose agents like Claude Code with composable, reusable capabilities.

## Core Architecture

### What Is a Skill?

At its simplest, a skill is a directory containing a `SKILL.md` file with required YAML frontmatter including `name` and `description` metadata.

### Progressive Disclosure Model

The system operates across three levels:

1. **First Level**: Metadata (name and description) loaded into the system prompt at startup, helping Claude recognize when to use each skill
2. **Second Level**: Full `SKILL.md` content loaded when Claude determines the skill is relevant
3. **Third Level+**: Additional bundled files referenced from `SKILL.md`, discoverable by Claude as needed

This approach keeps context lean while allowing effectively unbounded content through dynamic file loading.

## Practical Example: PDF Skill

The PDF skill structure includes:
- Core `SKILL.md` with primary guidance
- `reference.md` for general information
- `forms.md` for form-filling specific instructions

By organizing this way, "the skill author is able to keep the core of the skill lean, trusting that Claude will read forms.md only when filling out a form."

## Development Guidelines

### Best Practices

**Start with Evaluation**
Test agents on representative tasks to identify capability gaps, then build skills incrementally to address shortcomings.

**Structure for Scale**
When `SKILL.md` becomes unwieldy, split content into separate files. Keep mutually exclusive contexts separate to reduce token usage. Code serves both as executable tools and documentation—clarity about execution intent matters.

**Think from Claude's Perspective**
Monitor real-world skill usage and iterate based on observations. Pay special attention to skill `name` and `description`, as Claude uses these to decide whether triggering is appropriate.

**Iterate with Claude**
Collaborate with Claude to capture successful approaches and mistakes into reusable context and code within skills. Request self-reflection when trajectories diverge.

### Security Considerations

Skills provide new capabilities through instructions and code, creating potential vulnerability surfaces. The recommendation: "install skills only from trusted sources." When using less-trusted skills, audit bundled files thoroughly, examining code dependencies, resources, and any instructions directing Claude toward untrusted external network connections.

## Skills and Code Execution

Skills can bundle executable code Claude runs at its discretion. Large language models excel at many tasks, but certain operations suit traditional code better—sorting lists via tokens is expensive compared to running algorithms. More importantly, code provides deterministic reliability that token generation cannot match.

The PDF skill example includes a pre-written Python script extracting form fields without loading the script or PDF into context, making the workflow "consistent and repeatable."

## Current Availability

Agent Skills are currently supported across:
- Claude.ai
- Claude Code
- Claude Agent SDK
- Claude Developer Platform

## Future Direction

Upcoming features will support the full skill lifecycle: creation, editing, discovery, sharing, and usage.

Future exploration includes:
- Complementing MCP servers by teaching agents complex workflows involving external tools
- Enabling agents to "create, edit, and evaluate Skills on their own, letting them codify their own patterns of behavior into reusable capabilities"

## Resources

- **Documentation**: https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview
- **Cookbook**: https://github.com/anthropics/claude-cookbooks/tree/main/skills

## Key Takeaways

- Skills are directories with SKILL.md containing name and description metadata
- Progressive disclosure keeps context lean while allowing unbounded content
- Split large SKILL.md files into separate reference files
- Bundle executable code for deterministic, repeatable operations
- Only install skills from trusted sources
- Monitor real-world usage and iterate based on observations
