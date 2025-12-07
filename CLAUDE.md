# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository contains Claude Code skills, a knowledge base of Anthropic engineering articles, and placeholder directories for agents and hooks.

## Structure

```
claude-setup/
├── agents/           # Placeholder for agents (currently empty)
├── hooks/            # Placeholder for hooks (currently empty)
├── knowledgebase/    # Anthropic engineering articles on AI agents
│   ├── 01-effective-harnesses-for-long-running-agents.md
│   ├── 02-advanced-tool-use.md
│   ├── 03-code-execution-with-mcp.md
│   ├── 04-agent-skills.md
│   ├── 05-context-engineering.md
│   ├── 06-multi-agent-research-system.md
│   └── 07-claude-code-best-practices.md
└── skills/           # Claude Code skills
    ├── api-design-reviewer/   # API review for REST/GraphQL/gRPC
    ├── brand-guidelines/      # Anthropic brand colors and typography
    ├── canvas-design/         # Visual art in .png/.pdf with design philosophy
    ├── frontend-design/       # Production-grade frontend interfaces
    └── skill-creator/         # Meta-skill for creating new skills
```

## Knowledge Base

The `knowledgebase/` folder contains curated articles from Anthropic's engineering blog. Key concepts:

- **Progressive Disclosure**: Load information in layers as needed (metadata → SKILL.md → bundled resources)
- **Token Efficiency**: Code APIs reduce tokens by 98.7%, tool search by 85%, programmatic calling by 37%
- **Orchestrator-Worker Pattern**: Lead agent delegates to specialized subagents for 90%+ performance gains
- **Just-in-Time Loading**: Maintain identifiers, load content dynamically instead of pre-loading
- **Thinking Modes**: Use "think", "think hard", "think harder", "ultrathink" for complex problems

## Skill Architecture

Each skill follows this pattern:
```
skill-name/
├── SKILL.md           # Required: YAML frontmatter (name, description) + instructions
├── scripts/           # Optional: Executable Python/Bash scripts
├── references/        # Optional: Documentation loaded into context as needed
└── assets/            # Optional: Templates, fonts, images used in output
```

## Key Commands

### Create a New Skill
```bash
python skills/skill-creator/scripts/init_skill.py <skill-name> --path <output-directory>
```

### Validate a Skill
```bash
python skills/skill-creator/scripts/quick_validate.py <path/to/skill-folder>
```

### Package a Skill
```bash
python skills/skill-creator/scripts/package_skill.py <path/to/skill-folder> [output-directory]
```
Creates a `.skill` file (zip format) after validation.

## Skill Development Rules

- **SKILL.md frontmatter** must have `name` (hyphen-case, max 64 chars) and `description` (max 1024 chars, no angle brackets)
- **Allowed frontmatter keys**: name, description, license, allowed-tools, metadata
- **Keep SKILL.md under 500 lines** - split detailed content into `references/` files
- **Delete unused example files** created by init_skill.py
- **No auxiliary docs** - no README.md, CHANGELOG.md, etc. SKILL.md is the single source of truth
