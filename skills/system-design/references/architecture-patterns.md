# Architecture Patterns

## Quick Decision Matrix

| If you need... | Use | Avoid |
|----------------|-----|-------|
| Simple app, small team, fast iteration | Monolith | Microservices |
| Independent scaling, large teams | Microservices | Monolith |
| Event-driven, async processing | Message queues | Synchronous calls |
| Mobile-optimized API | BFF (Backend for Frontend) | Generic API |
| Cost optimization, variable load | Serverless | Always-on servers |
| Maximum control, predictable load | Traditional servers | Serverless |

---

## Monolith

| Aspect | Details |
|--------|---------|
| **Best for** | Startups, MVPs, small-medium apps, small teams |
| **Pros** | Simple deployment, easy debugging, less operational overhead, faster development initially |
| **Cons** | Scaling entire app together, large codebase over time, team coupling |
| **When to use** | Starting out, team < 10, unclear domain boundaries, rapid iteration needed |
| **When to avoid** | Large teams needing independence, vastly different scaling needs per component |
| **Works well with** | Any web framework (Django, Rails, Express, FastAPI) |
| **iOS considerations** | Single API endpoint, simpler client integration |

### Modular Monolith (Recommended Starting Point)
```
app/
├── modules/
│   ├── users/
│   │   ├── routes.py
│   │   ├── services.py
│   │   └── models.py
│   ├── orders/
│   │   ├── routes.py
│   │   ├── services.py
│   │   └── models.py
│   └── payments/
│       ├── routes.py
│       ├── services.py
│       └── models.py
├── shared/
│   ├── database.py
│   └── auth.py
└── main.py
```

**Benefits**: Can extract to microservices later if needed, clear boundaries now.

---

## Microservices

| Aspect | Details |
|--------|---------|
| **Best for** | Large organizations, independent team scaling, complex domains |
| **Pros** | Independent deployment, tech flexibility, isolated failures, team autonomy |
| **Cons** | Operational complexity, network latency, distributed transactions, debugging difficulty |
| **When to use** | Large teams (50+), clear domain boundaries, different scaling needs |
| **When to avoid** | Small teams, MVPs, unclear requirements, limited DevOps capacity |
| **Works well with** | Kubernetes, Docker, API Gateway, Service mesh |
| **iOS considerations** | API Gateway as single entry point, handle partial failures gracefully |

### Microservices Patterns

| Pattern | Use Case |
|---------|----------|
| API Gateway | Single entry point, auth, rate limiting |
| Service mesh | Service-to-service communication |
| Saga | Distributed transactions |
| CQRS | Separate read/write models |
| Event sourcing | Audit trail, event replay |

### When to Split Monolith
- Team conflicts on same codebase
- Different scaling needs (10x users vs 10x orders)
- Different release cadences needed
- Clear bounded contexts identified

---

## Backend for Frontend (BFF)

| Aspect | Details |
|--------|---------|
| **Best for** | Multiple clients with different needs (iOS, Web, Admin) |
| **Pros** | Client-optimized APIs, reduces client complexity, independent evolution |
| **Cons** | Code duplication across BFFs, more services to maintain |
| **When to use** | iOS + Web with significantly different data needs, multiple client teams |
| **When to avoid** | Single client type, simple data requirements |

### BFF Architecture
```
┌─────────┐  ┌─────────┐  ┌─────────┐
│ iOS App │  │ Web App │  │ Admin   │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     ▼            ▼            ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│ iOS BFF │  │ Web BFF │  │Admin BFF│
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┼────────────┘
                  ▼
         ┌───────────────┐
         │ Core Services │
         └───────────────┘
```

### iOS BFF Benefits
- Mobile-optimized payloads (smaller, combined)
- Handle multiple service calls server-side
- Version independently from web
- iOS-specific caching strategies

---

## Serverless

| Aspect | Details |
|--------|---------|
| **Best for** | Variable load, event-driven, cost optimization |
| **Pros** | No server management, auto-scaling, pay-per-use, quick deployment |
| **Cons** | Cold starts, execution limits, vendor lock-in, debugging difficulty |
| **When to use** | Variable traffic, event processing, webhooks, scheduled tasks |
| **When to avoid** | Consistent high load, long-running processes, WebSocket needs |
| **Works well with** | AWS Lambda, Vercel, Cloudflare Workers, API Gateway |
| **iOS considerations** | Cold starts can affect perceived latency, consider for background tasks |

### Serverless Use Cases

| Good Fit | Poor Fit |
|----------|----------|
| Webhooks | WebSocket servers |
| Image processing | Long-running jobs |
| API endpoints | High-frequency calls |
| Scheduled tasks | Consistent load |
| Event handlers | Stateful applications |

### Cold Start Mitigation
- Provisioned concurrency (AWS)
- Keep functions warm with scheduled pings
- Optimize bundle size
- Use edge functions for latency-sensitive endpoints

---

## Event-Driven Architecture

| Aspect | Details |
|--------|---------|
| **Best for** | Async processing, decoupled services, event sourcing |
| **Pros** | Loose coupling, scalability, resilience, audit trail |
| **Cons** | Eventual consistency, debugging complexity, event ordering |
| **When to use** | Background jobs, notifications, analytics, cross-service communication |
| **When to avoid** | Simple CRUD, real-time consistency required |
| **Works well with** | Redis Pub/Sub, RabbitMQ, AWS SQS/SNS, Kafka |

### Event-Driven Patterns

| Pattern | Use Case | Tool |
|---------|----------|------|
| Pub/Sub | Broadcast events | Redis, SNS |
| Work queue | Background jobs | Celery, Bull, SQS |
| Event log | Audit, replay | Kafka, EventStore |

### Example: Order Processing
```
1. User places order → OrderCreated event
2. Payment service → Processes payment → PaymentCompleted event
3. Inventory service → Reserves stock → StockReserved event
4. Notification service → Sends confirmation → NotificationSent event
5. iOS receives push notification
```

---

## Message Queues

| Queue | Best For | Ordering | Persistence |
|-------|----------|----------|-------------|
| Redis | Simple pub/sub, caching | No guarantee | Optional |
| RabbitMQ | Complex routing, reliability | FIFO per queue | Yes |
| AWS SQS | Serverless, AWS ecosystem | FIFO optional | Yes |
| Kafka | High throughput, event log | Partition-level | Yes |

### Background Job Pattern
```
┌─────────┐     ┌─────────┐     ┌─────────┐
│   API   │────▶│  Queue  │────▶│ Worker  │
└─────────┘     └─────────┘     └─────────┘
     │                               │
     │ Immediate response            │ Async processing
     ▼                               ▼
  "Job queued"               Send email, process image,
                             sync data, etc.
```

---

## Architecture Decision Framework

### Start Here
```
Is this an MVP or new project?
├── Yes → Modular Monolith
└── No → Continue...

Do different components need independent scaling?
├── Yes → Consider Microservices
└── No → Modular Monolith

Do iOS and Web need significantly different APIs?
├── Yes → Add BFF layer
└── No → Shared API

Is traffic highly variable or event-driven?
├── Yes → Consider Serverless for those parts
└── No → Traditional servers
```

---

## Architecture Checklist

- [ ] Start simple (monolith) unless clear reason for microservices
- [ ] Define module boundaries even in monolith
- [ ] Plan for async processing (message queues)
- [ ] Consider BFF if iOS needs differ from web
- [ ] Evaluate serverless for variable workloads
- [ ] Design for failure (retries, circuit breakers)
- [ ] Plan observability (logging, tracing, metrics)
- [ ] Document architecture decisions
