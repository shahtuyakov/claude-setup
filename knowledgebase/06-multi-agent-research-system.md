# How We Built Our Multi-Agent Research System

Source: https://www.anthropic.com/engineering/multi-agent-research-system

## Overview

Anthropic's Research feature demonstrates how multiple Claude agents can collaborate to explore complex topics more effectively than single-agent systems. The engineering journey from prototype to production revealed critical lessons about system architecture, tool design, and prompt engineering.

## Key Benefits of Multi-Agent Architecture

### Open-ended Problem Solving
Research demands flexibility to pivot based on discoveries. Single-agent systems follow linear paths, while multi-agent setups enable autonomous decision-making across many turns, adapting approaches as intermediate findings emerge.

### Performance Gains Through Parallelization
Anthropic's internal evaluations show "a multi-agent system with Claude Opus 4 as the lead agent and Claude Sonnet 4 subagents outperformed single-agent Claude Opus 4 by 90.2% on our internal research eval."

Parallel exploration by specialized subagents handles breadth-first queries effectively.

### Token Efficiency Analysis
Three factors explained 95% of performance variance in the BrowseComp evaluation:
- **Token usage alone: 80%**
- Tool calls: secondary factor
- Model selection: secondary factor

Multi-agent systems require approximately 15 times more tokens than standard chat interactions.

## System Architecture

The research system uses an **orchestrator-worker pattern**:

1. A lead agent analyzes queries and develops strategies
2. Specialized subagents explore different aspects simultaneously
3. Each subagent iteratively gathers information using search tools
4. Results consolidate through a citation agent for proper attribution

This contrasts with static Retrieval Augmented Generation (RAG) by enabling dynamic, adaptive information discovery.

## Critical Prompting Principles

### 1. Understand Agent Behavior
Building simulations with exact prompts and tools revealed failure modes:
- Agents continuing past sufficiency
- Using verbose queries
- Selecting incorrect tools

### 2. Delegation Framework
Lead agents require specific instructions including:
- Objectives
- Output formats
- Tool guidance
- Clear task boundaries

Vague instructions lead to duplicate work and search inefficiency.

### 3. Effort Scaling
Embedded rules guide resource allocation:
- **Simple fact-finding**: 1 agent with 3-10 tool calls
- **Complex research**: 10+ subagents with divided responsibilities

### 4. Tool Design
Agent-tool interfaces demand clarity matching human-computer design principles. Quality tool descriptions with distinct purposes prove essential—poor descriptions send agents down wrong paths.

### 5. Self-Improvement Capability
Claude models excel at prompt engineering themselves. One tool-testing agent rewrote descriptions based on usage patterns, "resulted in a 40% decrease in task completion time for future agents."

### 6. Search Strategy
Agents benefit from starting broad, then narrowing focus—mirroring expert human research practices rather than defaulting to specific, narrow queries.

### 7. Extended Thinking Integration
Thinking mode serves as a controllable scratchpad for planning. Subagents use interleaved thinking after tool results to evaluate quality and refine queries.

### 8. Parallel Tool Execution
Introducing parallelization—spawning 3-5 subagents simultaneously plus 3+ parallel tool calls—"cut research time by up to 90% for complex queries."

## Evaluation Methodology

### Small-Scale Testing
Start with ~20 test cases representing real usage patterns. Early development shows dramatic impact from changes; small samples reveal clear effects.

### LLM-as-Judge Framework
Single LLM calls evaluating against rubrics:
- Factual accuracy
- Citation accuracy
- Completeness
- Source quality
- Tool efficiency

Scale effectively for free-form research outputs.

### Human Oversight Importance
Manual testing catches edge cases automated evaluations miss—including hallucinations, source selection biases, and system failures.

## Production Engineering Challenges

### State Management
Agents maintain state across many tool calls over extended periods. System failures cascade unpredictably.

**Solutions:**
- Durable execution
- Error handling
- Resumption capabilities rather than complete restarts

### Debugging Non-Determinism
Production tracing monitors decision patterns and interaction structures without examining conversation contents, maintaining user privacy while diagnosing failures systematically.

### Deployment Strategy
Rainbow deployments gradually shift traffic between versions, preventing disruption to running agents when updating complex, stateful systems.

### Architectural Bottlenecks
Current synchronous execution of subagents simplifies coordination but creates information flow bottlenecks. Future asynchronous execution would enable additional parallelism with added complexity in result coordination.

## Additional Practical Patterns

### End-State Evaluation
Judge whether agents achieved correct final states rather than validating specific processes—agents may take different valid paths.

### Long-Horizon Context Management
Store essential information in external memory; spawn fresh subagents with clean contexts while maintaining continuity through handoffs.

### Artifact Distribution
Enable subagents to persist outputs independently via external systems, passing lightweight references to coordinators, reducing token overhead from copying large outputs.

## Key Takeaways

- Multi-agent systems outperform single agents by 90%+ on research tasks
- Orchestrator-worker pattern with lead agent and specialized subagents
- Token usage accounts for 80% of performance variance
- Parallel execution cuts research time by up to 90%
- Start with broad searches, then narrow focus
- Use extended thinking for planning after tool results
- Rainbow deployments prevent disruption to running agents
- Judge end states, not processes
- Store essential info externally for long-horizon tasks
