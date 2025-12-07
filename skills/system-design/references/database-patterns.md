# Database Patterns

## Quick Decision Matrix

| If you need... | Use | Avoid |
|----------------|-----|-------|
| Relational data, ACID transactions | PostgreSQL | MongoDB |
| Simple relational, easy setup | MySQL/MariaDB | PostgreSQL (if simplicity needed) |
| Flexible schema, document storage | MongoDB | SQL databases |
| Key-value caching, sessions | Redis | SQL databases |
| Full-text search | Elasticsearch / PostgreSQL FTS | Basic SQL LIKE |
| iOS local storage | Core Data / SQLite | UserDefaults (for large data) |
| Time-series data | TimescaleDB / InfluxDB | General SQL |

---

## PostgreSQL

| Aspect | Details |
|--------|---------|
| **Best for** | Complex queries, data integrity, JSON + relational hybrid |
| **Pros** | ACID compliant, advanced features (JSONB, full-text search, arrays), excellent performance, extensible |
| **Cons** | More complex setup than MySQL, can be overkill for simple apps |
| **When to use** | Most web/mobile backends, complex data relationships, need JSON flexibility |
| **When to avoid** | Simple key-value needs, extreme write scaling (consider sharding/Citus) |
| **Works well with** | SQLAlchemy, Prisma, TypeORM, PostGIS (geo), TimescaleDB (time-series) |
| **iOS considerations** | Backend only; pair with Core Data on iOS for offline sync |

### PostgreSQL Best Practices
- Use JSONB for flexible fields within relational structure
- Create indexes on frequently queried columns
- Use connection pooling (PgBouncer) for high connections
- Enable `pg_stat_statements` for query analysis

---

## MySQL / MariaDB

| Aspect | Details |
|--------|---------|
| **Best for** | Traditional web apps, read-heavy workloads, simple relational data |
| **Pros** | Easy to set up, well-documented, excellent read performance, wide hosting support |
| **Cons** | Less advanced than PostgreSQL, weaker JSON support, some SQL standard deviations |
| **When to use** | Simple to medium complexity apps, when team knows MySQL, read-heavy workloads |
| **When to avoid** | Complex queries, need advanced PostgreSQL features |
| **Works well with** | Laravel, Rails, Django, most ORMs |
| **iOS considerations** | Backend only; similar offline sync patterns as PostgreSQL |

---

## MongoDB

| Aspect | Details |
|--------|---------|
| **Best for** | Flexible schemas, document storage, rapid prototyping |
| **Pros** | Schema flexibility, horizontal scaling, good for hierarchical data, fast development |
| **Cons** | No ACID across documents (until v4+), join limitations, can lead to data inconsistency |
| **When to use** | Content management, catalogs, when schema evolves frequently, logging/analytics |
| **When to avoid** | Financial data, complex relationships, transaction-heavy operations |
| **Works well with** | Mongoose, Node.js, Express |
| **iOS considerations** | Realm (MongoDB mobile) for offline-first with sync to MongoDB backend |

---

## Redis

| Aspect | Details |
|--------|---------|
| **Best for** | Caching, sessions, rate limiting, real-time leaderboards, pub/sub |
| **Pros** | Extremely fast (in-memory), versatile data structures, pub/sub support |
| **Cons** | Memory-limited, persistence options have tradeoffs, not primary database |
| **When to use** | Session storage, caching layer, rate limiting, queues, real-time features |
| **When to avoid** | Primary data storage, large datasets |
| **Works well with** | Any primary database, Sidekiq (Ruby), Bull (Node.js) |
| **iOS considerations** | Backend only; reduces latency for frequently accessed data |

### Redis Use Cases
- Session storage (fast auth checks)
- API response caching
- Rate limiting (sliding window)
- Real-time leaderboards (sorted sets)
- Pub/sub for real-time updates

---

## SQLite

| Aspect | Details |
|--------|---------|
| **Best for** | Embedded databases, iOS local storage, serverless |
| **Pros** | Zero config, file-based, full SQL support, excellent for mobile |
| **Cons** | No concurrent writes, not for high-traffic backends, limited users |
| **When to use** | iOS local database (via Core Data), embedded applications, serverless (Turso/Litestream) |
| **When to avoid** | Multi-user backends, high write concurrency |
| **Works well with** | Core Data (iOS), GRDB (Swift), better-sqlite3 (Node) |
| **iOS considerations** | Primary choice for local persistence, Core Data uses SQLite underneath |

---

## iOS Local Storage Comparison

| Option | Best For | Capacity | Complexity |
|--------|----------|----------|------------|
| UserDefaults | Small settings, flags | KB | Low |
| Keychain | Sensitive data (tokens, passwords) | KB | Medium |
| Core Data | Structured data, relationships, offline-first | GB | High |
| SQLite (direct) | Full SQL control, custom needs | GB | Medium |
| File system | Documents, media, large files | GB | Low |

---

## Caching Strategy

| Level | Tool | Cache What | TTL |
|-------|------|------------|-----|
| Application | Redis | Session, computed data | Minutes-hours |
| Database | Query cache | Frequent queries | Minutes |
| HTTP | CDN/Varnish | Static responses, images | Hours-days |
| iOS | NSCache/URLCache | API responses, images | Session/configured |

### Cache Invalidation Strategies
- **TTL-based**: Simple, eventual consistency
- **Event-based**: Invalidate on writes, more complex
- **Hybrid**: TTL with event-based invalidation for critical data

---

## Database Selection Checklist

- [ ] Data model defined (relational vs document)
- [ ] Estimated scale (users, data volume, queries/second)
- [ ] Transaction requirements identified
- [ ] Backup and recovery strategy planned
- [ ] iOS offline requirements considered (local DB needed?)
- [ ] Caching layer decided (Redis for hot data?)
- [ ] Connection pooling configured for backend
