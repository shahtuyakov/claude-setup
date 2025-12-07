# Effective Context Engineering for AI Agents

Source: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents

## Overview

Context engineering represents a paradigm shift in building with large language models. Rather than focusing solely on prompt optimization, this approach involves "thoughtfully curating what information enters the model's limited attention budget at each step."

## Key Concepts

### Context vs. Prompt Engineering
- **Prompt engineering** focuses on crafting effective instructions
- **Context engineering** manages the entire set of tokens during LLM inference—including system instructions, tools, external data, and message history

### The Attention Budget Challenge
LLMs operate under architectural constraints similar to human working memory. Research on "needle-in-a-haystack" benchmarking reveals that as context window length increases, "the model's ability to accurately recall information from that context decreases."

The transformer architecture creates n² pairwise token relationships, stretching attention capacity as context grows.

## Core Principles

### System Prompts
Effective system prompts strike a "Goldilocks zone" between extremes:
- **Too brittle**: Hardcoded complex logic that fragments and requires constant maintenance
- **Too vague**: High-level guidance lacking concrete signals for desired behavior

**Recommendation:** Organize prompts into distinct sections using XML tags or Markdown headers. Start minimal and add clarity based on failure modes during testing.

### Tools Design
Tools should be:
- Self-contained with clear intended use
- Minimal in quantity (avoid bloated toolsets)
- Token-efficient in information returns
- Unambiguous in decision-making

### Examples
Rather than exhaustive edge-case lists, curate "diverse, canonical examples that effectively portray the expected behavior." Examples serve as efficient communication—like pictures versus lengthy descriptions.

## Context Retrieval Strategies

### Just-in-Time Approach
Instead of pre-loading all relevant data, maintain lightweight identifiers (file paths, queries, links) and dynamically load information at runtime.

**Claude Code demonstrates this** with Bash commands like `head` and `tail` for analyzing large datasets without loading full objects into context.

**Benefits:**
- Storage efficiency through metadata signals
- Progressive disclosure enabling layer-by-layer discovery
- Self-managed working memory focusing on relevant subsets

### Hybrid Strategy
Optimal approaches combine upfront retrieval for speed with autonomous exploration. Claude Code pre-loads CLAUDE.md files while using glob and grep for just-in-time file discovery, "effectively bypassing the issues of stale indexing and complex syntax trees."

## Long-Horizon Task Techniques

### 1. Compaction
Summarize conversations approaching context limits and restart with compressed summaries.

**Implementation:**
- Preserve architectural decisions and critical details
- Discard redundant outputs
- Start by maximizing recall, then improve precision by eliminating superfluous content

### 2. Structured Note-Taking
Agents maintain persistent memory outside the context window—like to-do lists or NOTES.md files—tracking progress across complex tasks.

**Example:** Pokémon agents develop maps, remember achievements, and maintain strategic notes without explicit memory prompting.

### 3. Sub-Agent Architectures
Specialized agents handle focused tasks with clean context windows, returning condensed summaries (typically 1,000-2,000 tokens) to the lead agent.

**Benefits:** "Clear separation of concerns" with detailed search isolated from synthesis.

## Implementation Guidance

**Choose based on task characteristics:**
- **Compaction**: Back-and-forth conversation flows
- **Note-taking**: Iterative development with clear milestones
- **Sub-agents**: Complex research requiring parallel exploration

## Guiding Principle

Find "the smallest set of high-signal tokens that maximize the likelihood of your desired outcome."

As models improve, less prescriptive engineering becomes possible, but treating context as "a precious, finite resource will remain central to building reliable, effective agents."

## Key Takeaways

- Context engineering manages all tokens, not just prompts
- Longer contexts reduce recall accuracy—treat context as precious
- Use just-in-time loading instead of pre-loading all data
- Combine upfront retrieval with autonomous exploration
- For long tasks: use compaction, note-taking, or sub-agents
- Start minimal and add based on failure modes
- Examples are more efficient than exhaustive edge-case lists
