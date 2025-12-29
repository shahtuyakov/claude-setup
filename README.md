# Claude Agents Hub

A multi-agent orchestration framework for Claude Code. Build production software with 7 specialized AI agents that coordinate automatically through a Hub Architecture.

## Why This Framework?

| Feature | Benefit |
|---------|---------|
| **7 Specialist Agents** | Architect, Backend, Database, Designer, DevOps, Frontend, iOS |
| **15 Production Skills** | React, Swift, Node.js, PostgreSQL, Docker, and more |
| **Hub Architecture** | Solves subagent limitations with delegation requests |
| **Parallel Execution** | Frontend and iOS work simultaneously |
| **Token Efficient** | Progressive disclosure loads info only when needed |

## Architecture

```
                              ┌─────────────────┐
                              │      USER       │
                              └────────┬────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │    ARCHITECT    │
                              │   (Orchestrator)│
                              └────────┬────────┘
                                       │
                         Delegation Request (JSON)
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                            HUB                                    │
│                  (Main Conversation Thread)                       │
│                                                                   │
│   Processes delegation requests, manages execution order,         │
│   aggregates results, passes context between agents               │
└───────────────────────────────┬──────────────────────────────────┘
                                │
           ┌────────────────────┼────────────────────┐
           │                    │                    │
           ▼                    ▼                    ▼
    ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
    │  DATABASE   │      │   BACKEND   │      │  DESIGNER   │
    │   Agent     │      │    Agent    │      │   Agent     │
    └─────────────┘      └─────────────┘      └─────────────┘
           │                    │                    │
           │                    │                    │
           ▼                    ▼                    ▼
    ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
    │  FRONTEND   │ ←──→ │    iOS      │      │   DEVOPS    │
    │   Agent     │      │   Agent     │      │   Agent     │
    └─────────────┘      └─────────────┘      └─────────────┘
         (Can execute in parallel)
```

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/claude-setup.git
cd claude-setup
```

### 2. Copy to your project

Copy the `.claude` folder structure to the root of your project:

```bash
cp -r agents skills commands settings.local.json YOUR_PROJECT/.claude/
```

Your project structure should look like:

```
your-project/
├── .claude/
│   ├── agents/
│   ├── skills/
│   ├── commands/
│   └── settings.local.json
└── ... your code
```

### 3. Start using agents

In Claude Code, invoke agents by describing your task:

```
"I need a user authentication system with email/password login"
```

The Architect agent will:
1. Analyze requirements
2. Create an implementation plan
3. Delegate to Database → Backend → Frontend agents
4. Coordinate the full implementation

## Agents

| Agent | Domain | Skills |
|-------|--------|--------|
| **Architect** | Orchestration, planning | task-breakdown, system-design |
| **Database** | Schema, migrations, queries | database-patterns |
| **Backend** | APIs, auth, business logic | node-backend, api-design-reviewer |
| **Frontend** | React, Next.js, UI | react-patterns, frontend-design |
| **iOS** | SwiftUI, native features | swift-patterns |
| **Designer** | Design systems, theming | design-patterns, brand-guidelines |
| **DevOps** | Docker, CI/CD, deployment | devops-patterns |

## Skills

### Development
- `react-patterns` - Modern React & Next.js
- `swift-patterns` - iOS/SwiftUI development
- `node-backend` - Node.js APIs
- `database-patterns` - SQL, NoSQL, ORMs

### Architecture
- `system-design` - Technology selection
- `api-design-reviewer` - API best practices
- `task-breakdown` - Feature decomposition
- `agent-orchestration` - Multi-agent workflows

### Design & DevOps
- `design-patterns` - Modern CSS, Tailwind, shadcn/ui
- `frontend-design` - Production UI creation
- `brand-guidelines` - Anthropic branding
- `devops-patterns` - Docker, CI/CD, cloud deployment

## Hub Architecture

The Hub Architecture solves a fundamental limitation: **Claude Code subagents cannot directly spawn other subagents.**

### How It Works

1. **Agent receives task** → Architect analyzes requirements
2. **Creates delegation request** → JSON specifying which agents to invoke
3. **Hub processes request** → Main conversation executes agents in order
4. **Results aggregate** → Hub combines outputs and passes context
5. **Returns to requestor** → Architect synthesizes final delivery

### Delegation Request Format

```json
{
  "type": "delegation_request",
  "agents": [
    {"name": "database", "task": "Create users table with auth fields"},
    {"name": "backend", "task": "Build auth endpoints", "dependsOn": ["database"]},
    {"name": "frontend", "task": "Create login form", "dependsOn": ["backend"]}
  ],
  "execution": "sequential"
}
```

## Common Workflows

| Project Type | Agent Sequence |
|--------------|----------------|
| Web App | Designer → Database → Backend → Frontend → DevOps |
| iOS App | Designer → Database → Backend → iOS → DevOps |
| Cross-Platform | Designer → Database → Backend → (Frontend + iOS) → DevOps |
| API Only | Database → Backend → DevOps |
| UI Refresh | Designer → Frontend |

## Technologies Covered

**Languages:** TypeScript, Swift, Python, SQL, GraphQL

**Frameworks:** React, Next.js, Express, NestJS, SwiftUI

**Databases:** PostgreSQL, MongoDB, SQLite, Redis, Prisma, Drizzle

**Frontend:** Tailwind CSS, shadcn/ui, Framer Motion, Radix UI

**DevOps:** Docker, Kubernetes, GitHub Actions, Vercel, Railway, AWS

## Contributing

Contributions welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

MIT License - see [LICENSE](LICENSE) for details.

---

Built for [Claude Code](https://claude.ai/code)
