# Contributing to Claude Agents Hub

Thanks for your interest in contributing! This guide will help you get started.

## Ways to Contribute

### 1. Add a New Skill

Skills are modular knowledge packages. To create one:

```bash
python skills/skill-creator/scripts/init_skill.py my-skill --path skills/
```

**Requirements:**
- `SKILL.md` must have valid YAML frontmatter with `name` and `description`
- Name: hyphen-case, max 64 characters
- Description: max 1024 characters, no angle brackets
- Keep SKILL.md under 500 lines
- Put detailed content in `references/` folder

**Validate your skill:**

```bash
python skills/skill-creator/scripts/quick_validate.py skills/my-skill
```

### 2. Add a New Agent

Agents are specialist AI personas. Create a new file in `agents/`:

```markdown
# Agent Name

## Role
One-line description of what this agent does.

## Skills
- skill-one
- skill-two

## Responsibilities
- What this agent handles
- When to delegate to this agent

## Delegation Protocol
How this agent creates delegation requests for other agents.
```

### 3. Improve Existing Skills

- Add new patterns to `references/` folders
- Update outdated code examples
- Fix errors in documentation
- Add missing edge cases

### 4. Report Issues

Open an issue for:
- Bugs in skills or agents
- Missing patterns or documentation
- Feature requests
- Questions about architecture

## Pull Request Process

1. **Fork** the repository
2. **Create a branch** for your feature: `git checkout -b feature/my-feature`
3. **Make changes** following the guidelines below
4. **Test** your changes work with Claude Code
5. **Commit** with clear messages: `git commit -m "Add X pattern to Y skill"`
6. **Push** to your fork: `git push origin feature/my-feature`
7. **Open a PR** with a clear description of changes

## Guidelines

### Skill Structure

```
skill-name/
├── SKILL.md           # Required: frontmatter + instructions
├── scripts/           # Optional: executable scripts
├── references/        # Optional: detailed documentation
└── assets/            # Optional: templates, images
```

### Code Style

- Use clear, descriptive names
- Include comments for complex logic
- Provide working code examples (not pseudocode)
- Test examples before submitting

### Documentation Style

- Be concise - developers skim documentation
- Lead with the most important information
- Use tables for comparisons
- Include code examples for every pattern

### What NOT to Include

- README.md or CHANGELOG.md in skills (SKILL.md is the single source)
- Time estimates or scheduling suggestions
- Redundant or duplicate patterns
- Untested code examples

## Skill Frontmatter Reference

```yaml
---
name: my-skill-name
description: Brief description of what this skill does (max 1024 chars)
license: MIT
allowed-tools:
  - Read
  - Write
  - Bash
metadata:
  version: "1.0"
  author: Your Name
---
```

## Questions?

Open an issue with the `question` label and we'll help you out.
