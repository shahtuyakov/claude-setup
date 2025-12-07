---
name: task-breakdown
description: Break down complex development requests into discrete, actionable tasks. Use when analyzing new features, integrations, refactoring requests, or any multi-component work that needs to be sequenced across agents (database, backend, frontend). Provides patterns for identifying components, dependencies, and complexity.
---

# Task Breakdown

Break complex requests into actionable tasks for agent delegation.

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
| UI, screens, user interaction | Frontend |
| Data + API | Database → Backend |
| API + UI | Backend → Frontend |
| Full stack | Database → Backend → Frontend |

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

Database:
- [ ] [specific task]

Backend:
- [ ] [specific task]

Frontend:
- [ ] [specific task]

Dependencies:
- [what depends on what]
```

## Sequencing Rules

1. **Database first** - Schema must exist before backend uses it
2. **Backend before frontend** - API must exist before UI calls it
3. **Auth early** - Authentication often blocks other features
4. **Core before extras** - Basic CRUD before advanced features

## Breaking Down Large Requests

If request is too large (complex):

1. Identify the **core MVP** - minimum viable version
2. List **enhancements** as separate tasks
3. Each task should be completable in one session

**Example:**
```
User request: "Build a social media app"

Split into:
- task-001: User registration and auth
- task-002: User profile CRUD
- task-003: Post creation and feed
- task-004: Follow/unfollow system
- task-005: Notifications
...
```

## Common Patterns

For pre-analyzed breakdowns of common features, see `references/common-features.md`.

## Output

Present breakdown to user:

```
Breaking down: [request]

Complexity: [simple/medium/complex]
Components: [database, backend, frontend]

Tasks:
1. Database: [1 line]
2. Backend: [1 line]
3. Frontend: [1 line]

Proceed with task-001?
```
