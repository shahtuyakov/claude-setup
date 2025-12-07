---
name: database-patterns
description: Database design and implementation patterns for modern applications. Use when designing schemas, writing migrations, optimizing queries, and configuring ORMs. Covers PostgreSQL, MongoDB, Prisma, Drizzle, indexing strategies, and security best practices.
---

# Database Patterns

Modern database design and implementation patterns.

## Database Selection

| Database | Best For | Use When |
|----------|----------|----------|
| PostgreSQL | Relational data, complex queries | Structured data, ACID needed, analytics |
| MongoDB | Document-oriented, flexible schema | Rapid iteration, nested data, horizontal scale |
| SQLite | Embedded, local-first | Mobile apps, desktop apps, edge |
| Redis | Caching, sessions | High-speed reads, ephemeral data |

## ORM Selection

| ORM | Best For | Trade-offs |
|-----|----------|------------|
| Drizzle | Performance, serverless | SQL knowledge required |
| Prisma | Developer experience | Larger bundle, slower edge |
| TypeORM | NestJS, decorators | Legacy patterns |
| Kysely | Type-safe SQL builder | Lower-level |

## Reference Files

| Topic | Load | Use When |
|-------|------|----------|
| Schema design | `references/schema-design.md` | Designing tables, relationships |
| PostgreSQL | `references/postgresql-patterns.md` | PostgreSQL-specific patterns |
| MongoDB | `references/mongodb-patterns.md` | MongoDB-specific patterns |
| ORM patterns | `references/orm-patterns.md` | Prisma, Drizzle usage |
| Migrations | `references/migrations.md` | Schema versioning |
| Security | `references/security.md` | SQL injection, encryption |

## Quick Start Patterns

### PostgreSQL Table

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    email_verified_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ
);

CREATE INDEX users_email_idx ON users(email) WHERE deleted_at IS NULL;
```

### Drizzle Schema

```typescript
import { pgTable, uuid, varchar, timestamp } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  name: varchar('name', { length: 100 }).notNull(),
  passwordHash: varchar('password_hash', { length: 255 }).notNull(),
  emailVerifiedAt: timestamp('email_verified_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
  deletedAt: timestamp('deleted_at', { withTimezone: true }),
});
```

### Prisma Schema

```prisma
model User {
  id              String    @id @default(uuid())
  email           String    @unique
  name            String
  passwordHash    String    @map("password_hash")
  emailVerifiedAt DateTime? @map("email_verified_at")
  createdAt       DateTime  @default(now()) @map("created_at")
  updatedAt       DateTime  @updatedAt @map("updated_at")
  deletedAt       DateTime? @map("deleted_at")

  posts           Post[]

  @@map("users")
}
```

### MongoDB Document

```typescript
interface User {
  _id: ObjectId;
  email: string;
  name: string;
  passwordHash: string;
  emailVerifiedAt?: Date;
  createdAt: Date;
  updatedAt: Date;
  deletedAt?: Date;
  // Embedded data
  profile?: {
    bio: string;
    avatarUrl: string;
  };
}
```

## Schema Design Principles

1. **Normalize first** - Start with 3NF, denormalize for performance
2. **UUID for IDs** - Better for distributed systems
3. **Timestamps always** - created_at, updated_at on every table
4. **Soft deletes** - deleted_at over hard deletes
5. **Constraints** - Use database constraints, not just app validation

## Index Guidelines

| Index Type | Use For |
|------------|---------|
| B-tree (default) | Equality, range queries |
| Hash | Equality only (rare) |
| GIN | Arrays, JSONB, full-text |
| GiST | Geometric, range types |
| BRIN | Large sorted tables |

## Common Patterns

| Pattern | Description |
|---------|-------------|
| Soft delete | `deleted_at` column instead of DELETE |
| Audit log | Separate table tracking all changes |
| Polymorphic | `type` column + type-specific columns |
| EAV | Entity-Attribute-Value (avoid if possible) |
| Materialized view | Pre-computed query results |

## Security Checklist

- [ ] Parameterized queries only
- [ ] Passwords hashed with bcrypt/argon2
- [ ] PII encrypted at rest
- [ ] Database user has minimal privileges
- [ ] SSL/TLS for connections
- [ ] No sensitive data in logs
