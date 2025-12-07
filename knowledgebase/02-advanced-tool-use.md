# Advanced Tool Use

Source: https://www.anthropic.com/engineering/advanced-tool-use

## Overview

Anthropic has released three beta features enabling Claude to discover, learn, and execute tools dynamically for more sophisticated AI agent workflows.

## The Three Features

### 1. Tool Search Tool
Allows Claude to discover tools on-demand rather than loading all definitions upfront.

**Impact:**
- Token consumption reduced from ~77K to 8.7K tokens (85% reduction)
- Opus 4.5 improved from 79.5% to 88.1% accuracy on MCP evaluations

**Implementation:** Mark tool definitions with `defer_loading: true` for on-demand discovery.

### 2. Programmatic Tool Calling
Enables Claude to write Python code orchestrating multiple tools within a sandboxed execution environment.

**Impact:**
- Token consumption dropped from 43,588 to 27,297 tokens (37% reduction)
- Latency improvements from eliminating 19+ inference passes
- Intermediate data stays in execution environment, not Claude's context

**Implementation:** Tools include `allowed_callers: ["code_execution_20250825"]`.

### 3. Tool Use Examples
Provides concrete usage patterns through sample tool calls in definitions.

**Impact:**
- Accuracy improved from 72% to 90% on complex parameter handling

**Implementation:** Add via `input_examples` property with realistic data showing minimal, partial, and full specification patterns.

## When to Use Each Feature

### Tool Search Tool
Most beneficial when:
- Tool definitions consume >10K tokens
- 10+ available tools experiencing selection accuracy issues

### Programmatic Tool Calling
Best for:
- Processing large datasets needing only summaries
- Multi-step workflows with 3+ dependent calls
- Parallel operations across many items

### Tool Use Examples
Most valuable for:
- Complex nested structures
- Many optional parameters where inclusion patterns matter
- APIs with domain-specific conventions

## Synergy

The features work together:
- **Tool Search** ensures correct tool discovery
- **Programmatic Tool Calling** ensures efficient execution
- **Tool Use Examples** ensure correct invocation

## Key Takeaways

- Use deferred loading for large tool libraries to reduce token consumption
- Programmatic tool calling keeps intermediate results out of context
- Concrete examples dramatically improve parameter handling accuracy
- Combine all three features for optimal performance
