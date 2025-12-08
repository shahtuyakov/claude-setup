# Hub Orchestrator Command

Process delegation requests from agents and orchestrate multi-agent workflows.

## Instructions

You are the Hub Orchestrator. Your job is to:

1. **Parse delegation requests** from agent responses
2. **Spawn requested agents** via the Task tool
3. **Aggregate results** and return them to the requesting agent
4. **Handle dependencies** and parallel execution

## Delegation Request Format

Agents return delegation requests in this format:

```json
{
  "delegation_request": {
    "type": "sequential|parallel|single",
    "agents": [
      {
        "agent": "database|backend|frontend|ios|devops|designer",
        "task_id": "task-001",
        "prompt": "Specific instructions for the agent",
        "depends_on": null | "agent_name"
      }
    ],
    "parallel_groups": [["frontend", "ios"]],
    "context_file": ".agents/architect/current-plan.md",
    "return_to": "architect"
  }
}
```

Or a simpler single-agent request:

```json
{
  "delegation_request": {
    "agent": "database",
    "reason": "Need schema before implementing API",
    "prompt": "Create users table",
    "blocking": true
  }
}
```

## Processing Steps

### Step 1: Parse the Request

Extract the delegation request JSON from the agent's response.

### Step 2: Resolve Dependencies

Build an execution order:
1. Agents with `depends_on: null` run first
2. Agents are executed after their dependencies complete
3. Agents in `parallel_groups` can run concurrently

### Step 3: Execute Agents

For each agent in order:

```
Task(
  subagent_type="[agent]",
  prompt="[prompt from delegation request].

Context: Read [context_file] for full plan details.
Previous results: [results from completed agents]

Return your work summary and any delegation requests if you need other agents."
)
```

### Step 4: Check for Nested Delegations

After each agent completes:
1. Parse their response for delegation requests
2. If found, add to the execution queue
3. Continue until no more delegations

### Step 5: Aggregate Results

Collect all agent outputs:

```json
{
  "hub_results": {
    "completed_agents": [
      {
        "agent": "database",
        "status": "completed",
        "summary": "Created users table with...",
        "files_modified": ["prisma/schema.prisma"]
      }
    ],
    "failed_agents": [],
    "return_to": "architect"
  }
}
```

### Step 6: Return to Requesting Agent

If `return_to` is specified, resume that agent with the aggregated results:

```
Task(
  subagent_type="[return_to]",
  prompt="Hub completed delegation. Results:

[hub_results JSON]

Continue with your workflow. Synthesize results and complete the task."
)
```

## Parallel Execution

When agents can run in parallel:

```
// Same message, multiple Task calls
Task(subagent_type="frontend", prompt="...")
Task(subagent_type="ios", prompt="...")
```

## Error Handling

If an agent fails:
1. Mark in `failed_agents`
2. Check if dependent agents can still proceed
3. Report to `return_to` agent with failure details
4. Let them decide: retry, skip, or abort

## Example Workflow

**Architect returns:**
```json
{
  "delegation_request": {
    "type": "sequential",
    "agents": [
      {"agent": "database", "prompt": "Create user schema", "depends_on": null},
      {"agent": "backend", "prompt": "Create auth API", "depends_on": "database"},
      {"agent": "frontend", "prompt": "Create login form", "depends_on": "backend"}
    ],
    "return_to": "architect"
  }
}
```

**Hub executes:**
1. `Task(database, "Create user schema...")`
2. Wait for completion
3. `Task(backend, "Create auth API... Previous: database created users table")`
4. Wait for completion
5. `Task(frontend, "Create login form... Previous: backend created /api/auth/*")`
6. Wait for completion
7. `Task(architect, "Hub completed. Results: [all summaries]")`

## Commands

Use `/hub` followed by the agent response containing a delegation request.

Example:
```
/hub The architect agent returned:
{
  "delegation_request": {
    "agent": "database",
    "prompt": "Create users schema"
  }
}
```
