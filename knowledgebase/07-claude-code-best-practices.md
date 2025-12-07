# Claude Code: Best Practices for Agentic Coding

Source: https://www.anthropic.com/engineering/claude-code-best-practices

## Overview

Claude Code is Anthropic's command-line tool for agentic coding, designed as a low-level, unopinionated platform providing near-raw model access. The tool enables engineers to integrate Claude into their workflows across various codebases and languages.

## 1. Customize Your Setup

### CLAUDE.md Files
Create special configuration files that Claude automatically incorporates into conversations. These should document:
- Bash commands and usage
- Core files and utility functions
- Code style guidelines
- Testing procedures
- Repository conventions
- Developer environment setup
- Project-specific warnings

**Placement options:**
- Repository root
- Parent directories (for monorepos)
- Child directories
- Home folder (`~/.claude/CLAUDE.md`) for global application

### Tool Permissions
Manage which tools Claude can execute through four mechanisms:
1. Select "Always allow" during prompts
2. Use the `/permissions` command to configure allowlists
3. Edit `.claude/settings.json` or `~/.claude.json` directly
4. Apply `--allowedTools` CLI flags for session-specific access

### Tool Enhancement
- Install the GitHub CLI (`gh`) for seamless GitHub integration
- Document custom bash tools within CLAUDE.md
- Configure MCP servers in project or global configs
- Create custom slash commands in `.claude/commands` folders using `$ARGUMENTS` for parameters

## 2. Provide Effective Context

### Information Delivery Methods
- Paste or drag-drop screenshots and design mocks
- Use tab-completion to reference files and folders
- Paste URLs for Claude to fetch and analyze
- Pipe data directly into Claude Code (logs, CSVs)
- Request that Claude pull data via bash or MCP tools

### Instruction Clarity
Specificity significantly improves outcomes.

**Instead of:** "add tests"
**Use:** "write test cases covering the scenario where users are logged out, avoiding mocks"

## 3. Common Workflows

### Explore, Plan, Code, Commit
1. Ask Claude to read relevant files without writing code
2. Request a plan using thinking modes ("think," "think hard," "think harder," "ultrathink")
3. Have Claude implement the solution
4. Request commit and pull request creation

### Test-Driven Development
1. Write tests first based on expected inputs/outputs
2. Confirm tests fail before implementation
3. Commit tests
4. Have Claude write code to pass tests
5. Verify with independent agents to prevent overfitting
6. Commit the implementation

### Visual Iteration
1. Enable screenshot capabilities (Puppeteer, iOS simulator)
2. Provide design mocks
3. Have Claude implement and iterate based on visual feedback

### Safe YOLO Mode
Use `--dangerously-skip-permissions` in containerized environments without internet access for uninterrupted autonomous work.

### Codebase Q&A
Leverage Claude for onboarding and exploration, asking questions about:
- Logging systems
- API endpoints
- Code patterns
- Architectural decisions

### Git and GitHub Integration
- Search commit history to answer design questions
- Have Claude write contextual commit messages
- Manage complex operations (rebasing, conflict resolution)
- Create pull requests and implement code review feedback
- Triage and categorize issues

## 4. Workflow Optimization

### Course Correction Tools
- Request planning before coding and verify approach
- Press **Escape** to interrupt and redirect
- **Double-tap Escape** to edit previous prompts and explore alternatives
- Ask Claude to undo changes and try different approaches

### Context Management
Use `/clear` between tasks to maintain focus and prevent performance degradation from irrelevant conversation history.

### Complex Tasks
Employ checklists and scratchpads for multi-step processes. Have Claude generate markdown checklists of issues, then address each systematically.

### Extended Thinking
Use thinking modes to allocate additional computation time for evaluation and planning on complex problems:
- "think"
- "think hard"
- "think harder"
- "ultrathink"

## 5. Advanced Patterns

### Multi-Claude Workflows
- Run parallel instances with separate contexts for verification
- Create multiple git checkouts for independent tasks
- Use git worktrees for lightweight, isolated development
- Establish separate working directories with consistent naming

### Headless Mode Automation
Use `-p` flag for non-interactive contexts like CI, pre-commit hooks, and automation:
- Issue triage triggered by GitHub events
- Subjective code reviews beyond traditional linting

### Fanning Out and Pipelining
- Generate task lists for migrations and batch processing
- Call Claude programmatically with `claude -p "<prompt>"` for pipeline integration
- Use `--output-format stream-json` for structured output

## 6. Jupyter Notebook Integration

Researchers can open Jupyter files alongside Claude Code for rapid exploration and iteration. Request aesthetic improvements to visualizations and formatting before sharing.

## Technical Configuration

### MCP Setup
Configure Model Context Protocol servers in three ways:
1. Project-specific configuration
2. Global configuration for all projects
3. Checked-in `.mcp.json` files for team sharing

### Custom Slash Commands
Store prompt templates in `.claude/commands` with special `$ARGUMENTS` keyword for parameterized execution.

## Key Takeaways

- Use CLAUDE.md files at various levels to provide context
- Be specific in instructions—vague requests yield poor results
- Follow the Explore → Plan → Code → Commit workflow
- Use TDD with independent verification agents
- Press Escape to interrupt, double-tap to edit previous prompts
- Use `/clear` between tasks to manage context
- Leverage thinking modes for complex problems
- Run parallel Claude instances for verification
- Use headless mode (`-p`) for automation and CI
- Configure MCP servers for extended capabilities
