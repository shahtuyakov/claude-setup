---
name: system-design
description: System design patterns and architectural decision frameworks for web applications and iOS native apps. Use when making technology choices, selecting architecture patterns, designing APIs, choosing databases, planning authentication, or architecting mobile app backends. Provides comparison matrices for informed decision-making.
---

# System Design

Architectural decision framework for web apps and iOS native applications with custom backends.

## Decision Process

1. Identify requirements (scale, platform, constraints)
2. Load relevant reference file (see table below)
3. Use comparison matrix to evaluate options
4. Verify compatibility using MCP context7 for latest docs
5. Document decision with rationale

## Reference Navigation

| Decision Type | Load | Use When |
|---------------|------|----------|
| API style | `references/api-patterns.md` | Choosing REST vs GraphQL vs gRPC |
| Database | `references/database-patterns.md` | Selecting primary database, caching layer |
| Authentication | `references/auth-patterns.md` | Designing auth flow, token strategy |
| Real-time | `references/realtime-patterns.md` | Adding live updates, push notifications |
| iOS architecture | `references/ios-patterns.md` | Native iOS app structure, offline-first |
| Backend architecture | `references/architecture-patterns.md` | Monolith vs microservices, serverless |
| File storage | `references/storage-patterns.md` | Media uploads, CDN, cloud storage |

## Decision Documentation

Record each decision in `.agents/architect/decisions.md`:

```
## [date] - [decision title]
Context: [what prompted this decision]
Options: [what was considered]
Decision: [what was chosen]
Rationale: [why]
```

## Platform Considerations

### Web + iOS Projects
- Design API with mobile constraints in mind (bandwidth, battery)
- Plan for offline-first on iOS with backend sync
- Consider BFF (Backend for Frontend) if needs diverge significantly

### iOS-Specific
- Always check `references/ios-patterns.md` for native patterns
- Auth must include Keychain storage and biometrics
- Push notifications require APNs backend integration
