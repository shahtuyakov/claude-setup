---
name: task-breakdown
description: Break down complex development requests into discrete, actionable tasks. Use when analyzing new features, integrations, refactoring requests, or any multi-component work that needs to be sequenced across agents. Provides patterns for identifying components, dependencies, and complexity.
---

# Task Breakdown

Break complex requests into actionable tasks for agent delegation.

## Available Agents

| Agent | Skill | Responsibilities |
|-------|-------|------------------|
| **Database** | database-patterns | Schema design, migrations, queries, ORMs |
| **Backend** | node-backend | APIs, business logic, authentication |
| **Frontend** | react-patterns | React/Next.js UI, state, forms |
| **iOS** | swift-patterns | SwiftUI apps, native iOS features |
| **DevOps** | devops-patterns | Docker, CI/CD, deployment, infra |
| **Designer** | design-patterns | Tokens, styling, themes, animations |

## Breakdown Process

1. **Identify scope** - What is being asked?
2. **List components** - Which agents are involved?
3. **Find dependencies** - What must happen first?
4. **Assess complexity** - Simple, medium, or complex?
5. **Sequence tasks** - Order for execution

## Component Identification

| If request involves... | Agent(s) needed |
|------------------------|-----------------|
| Data models, schema, tables | Database |
| API endpoints, business logic | Backend |
| Web UI, React components | Frontend |
| Native iOS app | iOS |
| Deployment, containers, CI/CD | DevOps |
| Styling, theming, design tokens | Designer |
| Data + API | Database → Backend |
| API + Web UI | Backend → Frontend |
| API + iOS app | Backend → iOS |
| Full web stack | Database → Backend → Frontend |
| Full iOS stack | Database → Backend → iOS |
| Full stack + deployment | Database → Backend → Frontend → DevOps |
| Styled components | Designer → Frontend |

## Complexity Assessment

| Complexity | Indicators | Approach |
|------------|------------|----------|
| **Simple** | Single component, clear requirements, <1 day work | One agent, direct |
| **Medium** | 2-3 components, some unknowns, 1-3 days work | Sequential agents |
| **Complex** | 4+ components, unclear scope, dependencies, >3 days | Split into multiple tasks |

## Task Template

```
# [task-id]: [short description]

Complexity: [simple/medium/complex]
Platform: [web/ios/both]

Database:
- [ ] [specific task]

Backend:
- [ ] [specific task]

Frontend: (if web)
- [ ] [specific task]

iOS: (if mobile)
- [ ] [specific task]

Designer: (if styling needed)
- [ ] [specific task]

DevOps: (if deployment needed)
- [ ] [specific task]

Dependencies:
- [what depends on what]
```

## Sequencing Rules

1. **Database first** - Schema must exist before backend uses it
2. **Backend before clients** - API must exist before web/iOS calls it
3. **Designer before Frontend** - Tokens/themes before styled components
4. **Auth early** - Authentication often blocks other features
5. **Core before extras** - Basic CRUD before advanced features
6. **DevOps last** - Deployment after features are complete
7. **Frontend & iOS parallel** - Can work simultaneously once API exists

## Breaking Down Large Requests

If request is too large (complex):

1. Identify the **core MVP** - minimum viable version
2. List **enhancements** as separate tasks
3. Each task should be completable in one session

**Example:**
```
User request: "Build a social media app with web and iOS"

Split into:
- task-001: Design system setup (Designer)
- task-002: User registration and auth (Database → Backend)
- task-003: User profile CRUD (Database → Backend → Frontend + iOS)
- task-004: Post creation and feed (Database → Backend → Frontend + iOS)
- task-005: Follow/unfollow system (Database → Backend → Frontend + iOS)
- task-006: Push notifications (Backend → iOS)
- task-007: CI/CD and deployment (DevOps)
...
```

## Common Patterns

For pre-analyzed breakdowns of common features, see `references/common-features.md`.

## Output

Present breakdown to user:

```
Breaking down: [request]

Complexity: [simple/medium/complex]
Platform: [web/ios/both]
Agents: [database, backend, frontend, ios, designer, devops]

Tasks:
1. Designer: [setup tokens/theme]
2. Database: [schema work]
3. Backend: [API endpoints]
4. Frontend: [web UI] (parallel with iOS)
5. iOS: [native app]
6. DevOps: [deployment]

Sequence: Designer → Database → Backend → Frontend + iOS → DevOps

Proceed with task-001?
```

## Common Flows

### Web App
```
Designer → Database → Backend → Frontend → DevOps
```

### iOS App
```
Designer → Database → Backend → iOS → DevOps
```

### Full Cross-Platform
```
Designer → Database → Backend → Frontend + iOS (parallel) → DevOps
```

### API Only
```
Database → Backend → DevOps
```

### UI Refresh
```
Designer → Frontend
```
