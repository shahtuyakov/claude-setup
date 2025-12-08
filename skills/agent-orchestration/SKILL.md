---
name: agent-orchestration
description: Hub orchestration patterns for multi-agent workflows. Use when processing delegation requests from agents, coordinating sequential/parallel agent execution, and aggregating results. Provides the protocol for agent-to-agent communication through the hub.
---

# Agent Orchestration Skill

## Overview

This skill provides patterns for orchestrating multi-agent workflows using the Hub Architecture. In this pattern:

- **Subagents cannot directly spawn other subagents**
- **The main conversation acts as a hub** that routes requests between agents
- **Agents communicate via structured delegation requests**

## Delegation Request Protocol

### Single Agent Request

When an agent needs help from one other agent:

```json
{
  "delegation_request": {
    "agent": "database",
    "reason": "Need users schema before implementing auth",
    "prompt": "Create users table with id, email, password_hash, created_at",
    "blocking": true
  }
}
```

| Field | Description |
|-------|-------------|
| `agent` | Target agent: database, backend, frontend, ios, devops, designer |
| `reason` | Why this delegation is needed |
| `prompt` | Specific instructions for the target agent |
| `blocking` | If true, wait for completion before continuing |

### Multi-Agent Request

When orchestrating multiple agents (typically from architect):

```json
{
  "delegation_request": {
    "type": "sequential",
    "agents": [
      {
        "agent": "database",
        "task_id": "auth-001",
        "prompt": "Create users and sessions tables",
        "depends_on": null
      },
      {
        "agent": "backend",
        "task_id": "auth-001",
        "prompt": "Implement auth endpoints",
        "depends_on": "database"
      },
      {
        "agent": "frontend",
        "task_id": "auth-001",
        "prompt": "Create login/register forms",
        "depends_on": "backend"
      },
      {
        "agent": "ios",
        "task_id": "auth-001",
        "prompt": "Create auth screens",
        "depends_on": "backend"
      }
    ],
    "parallel_groups": [
      ["frontend", "ios"]
    ],
    "context_file": ".agents/architect/current-plan.md",
    "return_to": "architect"
  }
}
```

| Field | Description |
|-------|-------------|
| `type` | `sequential`, `parallel`, or `mixed` |
| `agents` | Array of agent tasks |
| `depends_on` | Agent that must complete first (null = no dependency) |
| `parallel_groups` | Arrays of agents that can run concurrently |
| `context_file` | Shared context file for all agents (JSON format: `.agents/architect/current-plan.json`) |
| `return_to` | Agent to receive aggregated results |

## Hub Processing

When the hub receives a delegation request:

### Step 1: Parse Request

Extract the `delegation_request` JSON from the agent's response.

### Step 2: Build Execution Graph

```
database (no deps) ──┐
                     ├──► backend ──┬──► frontend
                     │              │
                     │              └──► ios (parallel with frontend)
                     │
designer (no deps) ──┘
```

### Step 3: Execute in Order

1. Run agents with no dependencies first
2. Pass results to dependent agents
3. Run parallel groups concurrently
4. Collect all results

### Step 4: Spawn Agents

```
Task(
  subagent_type="database",
  prompt="Task auth-001: Create users and sessions tables.

Read: .agents/architect/current-plan.json

Return summary under 500 tokens including:
- Tables/schemas created
- Key decisions
- Notes for dependent agents"
)
```

### Step 5: Pass Context Forward

```
Task(
  subagent_type="backend",
  prompt="Task auth-001: Implement auth endpoints.

Read: .agents/architect/current-plan.json

Previous agent results:
- database: Created users(id, email, password_hash) and sessions(id, user_id, token) tables

Return summary under 500 tokens."
)
```

### Step 6: Handle Parallel Execution

When agents can run in parallel, spawn them in the same message:

```
// Both run concurrently
Task(subagent_type="frontend", prompt="...")
Task(subagent_type="ios", prompt="...")
```

### Step 7: Aggregate Results

```json
{
  "hub_results": {
    "task_id": "auth-001",
    "completed_agents": [
      {
        "agent": "database",
        "status": "completed",
        "summary": "Created users and sessions tables",
        "files": ["prisma/schema.prisma"]
      },
      {
        "agent": "backend",
        "status": "completed",
        "summary": "Created POST /auth/login, /auth/register, /auth/logout",
        "files": ["src/routes/auth.ts"]
      }
    ],
    "failed_agents": [],
    "total_files_modified": 5
  }
}
```

### Step 8: Return to Requesting Agent

```
Task(
  subagent_type="architect",
  prompt="Hub completed delegation for task auth-001.

Results:
[hub_results JSON]

All agents completed successfully. Continue with synthesis."
)
```

## Handling Nested Delegations

Agents may return their own delegation requests:

```
backend returns:
{
  "work_completed": "Implemented auth endpoints",
  "delegation_request": {
    "agent": "database",
    "reason": "Need to add email_verified column",
    "prompt": "Add email_verified boolean to users table"
  }
}
```

The hub should:
1. Note the work completed
2. Process the nested delegation
3. Return combined results

## Error Handling

### Agent Failure

```json
{
  "hub_results": {
    "completed_agents": [...],
    "failed_agents": [
      {
        "agent": "frontend",
        "status": "failed",
        "error": "Could not find component structure",
        "suggestion": "Run designer agent first"
      }
    ]
  }
}
```

### Dependency Failure

If a dependency fails, mark dependent agents as blocked:

```json
{
  "blocked_agents": [
    {
      "agent": "backend",
      "blocked_by": "database",
      "reason": "Database agent failed to create schema"
    }
  ]
}
```

## Workflow Examples

### Full Stack Feature

```
Architect → Hub:
  delegation_request:
    designer → database → backend → [frontend, ios] → devops

Hub executes:
  1. designer: Create tokens
  2. database: Create schema
  3. backend: Create API
  4. frontend + ios: Create UI (parallel)
  5. devops: Setup deployment
  6. Return to architect
```

### Quick Fix

```
Backend → Hub:
  delegation_request:
    agent: database
    prompt: "Add index on users.email"

Hub executes:
  1. database: Add index
  2. Return to backend
```

### Cross-Platform UI

```
Designer → Hub:
  delegation_request:
    parallel_groups: [[frontend, ios]]
    prompt: "Apply new button styles"

Hub executes:
  1. frontend + ios (parallel): Update buttons
  2. Return to designer
```

## Best Practices

1. **Keep prompts specific** - Include exact requirements, not vague goals
2. **Include context files** - Always reference shared plan documents
3. **Limit delegation depth** - Avoid deep nesting (max 2 levels)
4. **Use parallel when possible** - Frontend/iOS can often run together
5. **Handle failures gracefully** - Always check for failed agents
6. **Aggregate meaningfully** - Combine results into actionable summaries
