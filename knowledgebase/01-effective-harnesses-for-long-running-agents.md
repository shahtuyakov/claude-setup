# Effective Harnesses for Long-Running Agents

Source: https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents

## Overview

Anthropic has developed solutions to enable AI agents to work effectively across multiple context windows. The core challenge: agents operate in discrete sessions with no memory of previous work.

## The Two-Part Solution

### 1. Initializer Agent
Sets up the foundational environment on first run:
- Creates an `init.sh` script for running the development server
- Generates a `claude-progress.txt` file documenting work history
- Makes initial git commits showing added files

### 2. Coding Agent
Subsequent sessions focus on incremental progress while maintaining clean, mergeable code.

## Key Environmental Components

### Feature List Management
The system uses a structured JSON file containing 200+ features initially marked as "failing." The document specifies that agents should "only by changing the status of a passes field" and warns "It is unacceptable to remove or edit tests."

### Incremental Progress Approach
Rather than attempting complete implementation, agents work on single features per session. Git commits with descriptive messages and progress file summaries enable recovery from mistakes and maintain clarity.

### Testing Requirements
Agents initially marked features complete without proper testing. The solution involved explicit prompting to use browser automation tools (Puppeteer MCP) for end-to-end verification, mimicking human testing patterns.

## Session Startup Protocol

Each session begins with these steps:
1. Run `pwd` to confirm working directory
2. Read git logs and progress files
3. Select highest-priority incomplete feature
4. Run `init.sh` and verify basic functionality
5. Begin focused feature work

## Common Failure Modes & Solutions

| Problem | Initializer Solution | Coding Agent Solution |
|---------|-------|-------|
| Premature project completion | Comprehensive feature list | Work on one feature; only mark complete after testing |
| Undocumented progress | Git repo + progress file | Begin with documentation review; end with commits |
| Runtime confusion | `init.sh` script | Read init.sh before beginning work |

## Future Directions

Open questions remain regarding single-agent versus multi-agent architectures. Specialized agents (testing, QA, cleanup) might improve software development workflows. These principles may generalize to scientific research and financial modeling applications.

## Key Takeaways

- Use initializer agents to set up foundational environment
- Maintain progress files and git commits for session continuity
- Work on single features per session rather than attempting complete implementation
- Explicitly prompt agents to test their work using browser automation
- Start each session by reading documentation and progress files
