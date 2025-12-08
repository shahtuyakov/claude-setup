# Agent System Documentation

## Hub Architecture

This project uses a **Hub Architecture** for multi-agent orchestration. In this pattern:

```
                    ┌─────────────────────┐
                    │   Main Conversation │
                    │       (Hub)         │
                    └──────────┬──────────┘
                               │
           ┌───────────────────┼───────────────────┐
           │                   │                   │
           ▼                   ▼                   ▼
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │  Architect  │     │  Frontend   │     │     iOS     │
    └─────────────┘     └─────────────┘     └─────────────┘
           │                   │                   │
           │                   │                   │
           ▼                   ▼                   ▼
    Returns delegation   Returns delegation   Returns delegation
    requests to hub      requests to hub      requests to hub
```

### Why Hub Architecture?

Subagents **cannot directly spawn other subagents** - this is a limitation of the current system. The Hub Architecture works around this by:

1. **Agents return structured delegation requests** instead of spawning agents
2. **The hub (main conversation) processes these requests** and spawns the requested agents
3. **Results are aggregated and returned** to the requesting agent

## Available Agents

| Agent | Color | Skills | Purpose |
|-------|-------|--------|---------|
| **architect** | blue | node-backend, api-design-reviewer | Orchestrate multi-component tasks |
| **database** | green | database-patterns | Schema, migrations, queries |
| **backend** | red | node-backend, api-design-reviewer | APIs, business logic, auth |
| **frontend** | cyan | react-patterns, frontend-design | React/Next.js UI |
| **ios** | pink | swift-patterns | SwiftUI/iOS apps |
| **designer** | yellow | design-patterns | Tokens, styling, themes |
| **devops** | orange | devops-patterns | Docker, CI/CD, deployment |

## Delegation Request Protocol

### Single Agent Request

When an agent needs another agent's help:

```json
{
  "delegation_request": {
    "agent": "database",
    "reason": "Need schema before implementing API",
    "prompt": "Create users table with email, password_hash",
    "blocking": true
  }
}
```

### Multi-Agent Request (Architect)

When orchestrating multiple agents:

```json
{
  "delegation_request": {
    "type": "sequential",
    "agents": [
      {"agent": "database", "prompt": "Create schema", "depends_on": null},
      {"agent": "backend", "prompt": "Create API", "depends_on": "database"},
      {"agent": "frontend", "prompt": "Create UI", "depends_on": "backend"}
    ],
    "parallel_groups": [["frontend", "ios"]],
    "return_to": "architect"
  }
}
```

## Workflow Example

### 1. User Request
```
"Add user authentication"
```

### 2. Invoke Architect
```
Task(subagent_type="architect", prompt="Add user authentication to the app")
```

### 3. Architect Returns Delegation Request
```json
{
  "plan": "Implement auth with JWT...",
  "delegation_request": {
    "type": "sequential",
    "agents": [
      {"agent": "database", "prompt": "Create users table"},
      {"agent": "backend", "prompt": "Create auth endpoints"},
      {"agent": "frontend", "prompt": "Create login form"}
    ],
    "return_to": "architect"
  }
}
```

### 4. Hub Processes Request
```
Task(subagent_type="database", prompt="Create users table...")
// Wait for completion
Task(subagent_type="backend", prompt="Create auth endpoints. Database created: users table...")
// Wait for completion
Task(subagent_type="frontend", prompt="Create login form. Backend created: /api/auth/*...")
// Wait for completion
```

### 5. Hub Returns Results
```
Task(subagent_type="architect", prompt="Hub completed. Results:
- database: Created users table
- backend: Created auth endpoints
- frontend: Created login form
Continue with synthesis.")
```

### 6. Architect Synthesizes
```
"Authentication implemented:
- Database: users table with email, password_hash
- Backend: POST /auth/login, /auth/register
- Frontend: LoginForm, RegisterForm components"
```

## State Management

Each agent maintains state in `.agents/[agent]/`:

```
.agents/
├── state.json              # Current task, active agent
├── architect/
│   ├── current-plan.json   # Active task plan (JSON format)
│   ├── decisions.md        # Architectural decisions log
│   └── status.json         # Agent status
├── database/
│   ├── notes.md            # Schema decisions
│   └── status.json
├── backend/
│   ├── notes.md            # API decisions
│   └── status.json
└── ...
```

## Using the Hub Command

The `/hub` command processes delegation requests:

```
/hub The architect returned:
{
  "delegation_request": {
    "agent": "database",
    "prompt": "Create users schema"
  }
}
```

## Using the Skill

Load the `agent-orchestration` skill for patterns:

```
Skill(agent-orchestration)
```

## Common Sequences

```
Web App:        Designer → Database → Backend → Frontend → DevOps
iOS App:        Designer → Database → Backend → iOS → DevOps
Cross-Platform: Designer → Database → Backend → [Frontend + iOS] → DevOps
API Only:       Database → Backend → DevOps
UI Refresh:     Designer → [Frontend + iOS]
```

## Best Practices

1. **Always start with Architect** for multi-component tasks
2. **Use parallel groups** when agents don't depend on each other (frontend + ios)
3. **Keep prompts specific** - include exact requirements
4. **Reference context files** - `.agents/architect/current-plan.json`
5. **Handle failures gracefully** - check hub results for errors
6. **Limit nesting** - avoid more than 2 levels of delegation
